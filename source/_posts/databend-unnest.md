---
title: Databend Unnest 函数的实现
date: 2023-03-13 09:29:50
tags:
    - 数据库
categories:
    - 数据库
    - 源码笔记
---

`unnest` 是一个在分析查询中比较常见的函数，它的作用是将一个嵌套（多维）数组展开为一个一维向量。

<!-- more -->


```sql
select unnest([1,2,3]);
----
1
2
3

select unnest([[1,2],[3]]);
----
1
2
3
```

Tacking issue: https://github.com/datafuselabs/databend/issues/10295
在下文中，我会逐步拆解 Databend 中实现 unnest 函数所需要处理的要点。


## 基本思路

对于一般的函数，我们可以按照这个文档里的方法，通过统一的 `FunctionFactory` 进行注册。Databend 会通过统一抽象的 API 进行函数调用与向量化执行。

普通的函数的一行输入会对应一行输出，但是 `unnest` 并不是，它的一行输入会对应多行输出，这种函数也被称作 Set Returning Function (SRF)。所以我们并不能将 `unnest` 像普通函数一样通过 `FuntionFactory` 进行注册，而需要为其设计单独的逻辑。

参考 PostgreSQL 与 DuckDB 等数据库的实现可以发现，它们都将 `unnest` 函数重写为了一个单独的算子（PostgreSQL: `ProjectSet` 算子，DuckDB：`UNNEST` 算子）。因此，Databend 也沿用这种设计，单独抽离出一个 `Unnest` 算子进行相关计算。

`Unnest` 算子主要需要承载的功能是：接受 `DataBlock` 向量化输入，按照 `unnest` 列将输入的各行展开，最后将展开后的列组成 `DataBlock` 输出。但是这将会面临几个问题：

1. 如何将原本的 EvalScalar算子（更进一步说，其中的 FunctionCall）转换为 Unnest 算子？
2. 如何处理 unnest 函数与其他操作结合的情况？（如 select unnest([1,2,3]) + 1，输出三行：2,3,4）
3. 如何按照 unnest 列展开其他列？如何处理多个 unnest 列？

且看下文分解。

## 构建算子

上述的 1. 与 2. 其实可以归纳为同一个问题，那就是 Planner 如何为 `Unnest` 构建执行计划与算子。

### Databend 表达式执行简介

这里先简单说明一下 Databend 中表达式算子的构建与执行：

1. AST -> 逻辑计划。在 Bind 阶段，AST 中的表达式会被 semantic check 为 `ScalarExpr`，并被放入逻辑计划 `EvalScalar` 中。
2. 逻辑计划 -> 物理计划。在物理计划构建时，`ScalarExpr` 会被 expression check 为 `Expr`，并被放入物理计划 `EvalScalar` 中。`Expr` 是可以被 `Evaluator` 执行的结构。
3. 物理计划 -> 流水线算子。在构建流水线时，`EvalScalar` 会被构建为 `BlockOperator::Map` 算子。在流水线执行过程中，`Map` 算子会接受 `DataBlock` 并根据构建好的 Expr 进行计算。每一个 `Expr` 会输出一列数据，并添加到原始 `DataBlock` 的最后并输出到下游（不需要的列会被后续 `Projection` 算子移除）。

现在要在表达式计算中引入 `Unnest`，就在上述三个部分中做出修改。

### Unnest 表达式

对于构建逻辑计划，这里选择引入一个新的 `ScalarExpr`：`ScalarExpr::Unnest`。在 semantic check 时，可能会对特定函数调用进行重写，可以利用这个流程，将 `unnest` 函数调用重写为 ScalarExpr::Unnest 表达式（https://github.com/datafuselabs/databend/blob/583a3f2029d749b5974445fd90ad137dc0b2fc91/src/query/sql/src/planner/semantic/type_check.rs#L1704）。其结构如下：

```Rust
pub struct Unnest {
    pub argument: Box<ScalarExpr>,
    pub return_type: Box<DataType>,
}
```

后续的 `RawExpr`, `Expr`, `RemoteExpr` 结构可以直径从 `ScalarExpr` 转换得到。semantic check 也会保证 `Unnest` 的参数 `argument` 的返回类型一定是 `DataType::Array`。

P.S. `Unnest` 表达式依然是逻辑计划 `EvalScalar` 的一部分。


### Unnest 物理计划

接下来便是物理计划的构建。在上一节“基本思路”中提到，`Unnest` 是一种 SRF，与其他表达式的执行逻辑差别很大，需要单独设计一种算子，因此这里引入一个新的物理计划 `Unnest`。

```Rust
pub struct Unnest {
    /// A unique id of operator in a `PhysicalPlan` tree.
    /// Only used for display.
    pub plan_id: u32,
    pub input: Box<PhysicalPlan>,
    /// How many unnest columns.
    pub num_columns: usize,
    /// Only used for explain
    pub stat_info: Option<PlanStatsInfo>,
}
```

这里有一个取巧的地方，就是 `Unnest` 中只记录了有多少列 unnest 列 `num_columns`，可以这样做的原因稍后会解释。

在将逻辑计划 `EvalScalar` build 为物理计划 `EvalScalar` 时，如果表达式中存在 `Unnest` 表达式，我们需要把他们提取出来作为单独一层的物理计划。由于 `Unnest` 表达式需要能够与其他表达式一起嵌套使用，所以在构建物理计划的时候我们以 `Unnest` 为分界，将表达式计算的物理计划分成三层（编号为执行顺序）：

```
3. Eval After Unnest Scalar
    |_______2. Unnest
            |_______1. Eval Before Unnest Scalar
```

每一层的列输入和列输出如下所示（编号与上述编号对应）：

```
1. [i1, i2, .., in] -> [i1, i2, .. in, b1, b2, .., bm]
2. [i1, i2, .. in, b1, b2, .., bm] -> [i1, i2, .. in, u1, u2, .., um]
3. [i1, i2, .. in, u1, u2, .., un] -> [i1, i2, .. in, u1, u2, .., um, o1, o2, .., op]
```

其中 n 为上游输入的列数，m 为 `Unnest` 列数，即 `Unnest` 内部参数需要执行的 `Expr`；p 为 `Unnest` 之后执行的表达式个数，即非 `Unnest` 函数内部参数需要执行的 `Expr`。

在真正执行时，计划 1. 会将需要执行 `Unnest` 的数据放到 `DataBlock` 后侧；计划 2. 会将 `Unnest` 参数列原地更新为展开后的 `Unnest` 列；计划 3. 将展开后的数据进行最后整体的表达式计算。

由于计划 2. 中需要执行 `Unnest` 的列一定在 `DataBlock` 的最后，所以我们只需要在 `Unnest` 计划中记录有多少 unnest 列即可。

在具体实现上，每一个 `ScalarExpr` 会被遍历两次，第一次会收集 `Unnest` 参数列，并转换成 `Expr`。第二次会将剩余的外层 `ScalarExpr` 转换成 `Expr`，在这个过程中，`Unnest` 子句会被替换为 `Expr::ColumnRef`，做为一个最底层数据列进行处理（和  Subquery 类似，因为这层 `EvalScalar` 能够直接拿到 `Unnest` 之后的数据，不用进行表达式计算了）。

两个此收集到的 `Expr` 会被分别构建为两个 `PhysicalPlan::EvalScalar`，分别为 `PhysicalPlan::Unnest` 的子计划与父计划。
具体代码：https://github.com/datafuselabs/databend/blob/583a3f2029d749b5974445fd90ad137dc0b2fc91/src/query/sql/src/executor/physical_plan_builder.rs#L305

### Unnest 流水线算子

之前的工作都是 `Unnest` 之外的准备工作，本节将会介绍 `Unnest` 算子的具体执行流程，也是 `Unnest` 函数的核心所在。

单独展开 `Unnest` 列比较简单，直接提取参数 `ArrayColumn` 中的底层数据即可。实现 `Unnest` 的要点在于如何跟随 `Unnest` 列一起展开非 `Unnest` 列以及如何处理多 `Unnest` 列。

Unnest 的核心我总结为“按行展开”。也就是对于 `Unnest` 列中的每一行 a（每一行是一个 `Array`），我们将其展开为多行，在原数据中的对应的行的其他列需要和展开后的 a 对齐。对于非 `Unnest` 列，通过复制自己的值进行对齐（replicated）；对于 `Unnest` 列，以展开后最长的 `Unnest` 列为准，使用 `NULL` 补齐。

举个例子：

```
+---------+--------+
| arr     | number |
+---------+--------+
| [1,2]   |      0 |
| [3,4,5] |      1 |
+---------+--------+

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

+-------------+--------+
| unnest(arr) | number |
+-------------+--------+
| 1           |      0 |
| 2           |      0 |
| 3           |      1 |
| 4           |      1 |
+-------------+--------+
```

```
+---------+---------+
| origin1 | origin2 |
+---------+---------+
| [1,2]   | [1,2,3] |
| [3,4,5] | [4,5]   |
+---------+---------+

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

+---------+---------+
| unnest1 | unnest2 |
+------ --+---------+
| 1       | 1       |
| 2       | 2       |
| NULL    | 3       |
| 3       | 4       |
| 4       | 5       |
| 5       | NULL    |
+---------+---------+
```

可见，展开后的行和展开前的行是对应的。

由于 Databend 是向量化执行，每一次计算会接受一个含有多行的 `DataBlock`，所以对 `Unnest` 的计算逻辑我们也最好对整个 `DataBlock` 进行一次性处理。
由于 Databend 中 `ArrayColumn` 的结构为：

```Rust
pub struct ArrayColumn<T: ValueType> {
    pub values: T::Column,
    pub offsets: Buffer<u64>,
}
```

整个 `Array` 列其实是存在一个连续的内存中的（每一行数据根据 `offsets` 进行 slice 得出），因此我们可以直接将底层 `Column` 作为输出，再根据 `offsets` 展开复制 `Unnest` 列的数据即可。但整个过程只能应用于单 `Unnest` 列的情况，多 `Unnest` 列我们仍然需要按行处理。

因此，`Unnest` 算子的计算流程总体上分为两步：

1. 处理所有 Unnest 列，得到对齐后的所有 `Unnest` 列以及统一的 `offsets`。（https://github.com/datafuselabs/databend/blob/583a3f2029d749b5974445fd90ad137dc0b2fc91/src/query/sql/src/evaluator/block_operator.rs#L235）
2. 将上一步计算得到的 `offsets` 应用于非 `Unnest` 列进行复制，得到最后可以输出的结果。（https://github.com/datafuselabs/databend/blob/583a3f2029d749b5974445fd90ad137dc0b2fc91/src/query/sql/src/evaluator/block_operator.rs#L161）

其中第 2. 步比较简单，这里就重点介绍一下第 1. 步的具体流程。

首先，我们可以根据所有 `Unnest` 列的 `offsets` 提前计算出最后的 `offsets` 数组。即对于每一行，在所有 offset 中取最大值。

```Rust
// 提前收集 ArrayColumn 中的每一行 Column
let unnest_columns = unnest_columns
    .into_iter()
    .map(|col| {
        let typ = col.values.data_type().unnest();
        let arrays = col.iter().map(|c| c.unnest()).collect::<Vec<_>>();
        (typ, arrays)
    })
    .collect::<Vec<_>>();
let mut offsets = Vec::with_capacity(num_rows + 1);
offsets.push(0);
let mut new_num_rows = 0; // Rows of the unnested `Column`s.
for row in 0..num_rows {
    let len = unnest_columns
        .iter()
        .map(|(_, col)| unsafe { col.get_unchecked(row).len() })
        .max()
        .unwrap();
    new_num_rows += len;
    offsets.push(new_num_rows);
}
```

然后，我们就可以开始按行遍历所有需要 unnest 的 `ArrayColumn`。如果当前行展开后的行数不满足结果需要的行数，则补 `NULL`（向 `validity` 这个 `Bitmap` 中填充 `false`）。为了充分利用编译器的向量化优化，我们应该尽量避免出现分支语句，又由于非 `NULL` 行和 `NULL` 行都是连续的，我们可以分别计算出它们所需要的行数并依次进行数据填充，从而避免使用分支语句。

```Rust
for (row, w) in offsets.windows(2).enumerate() {
    let len = w[1] - w[0];
    for ((typ, cols), builder) in unnest_columns.iter().zip(col_builders.iter_mut()) {
        let inner_col = unsafe { cols.get_unchecked(row) };
        let inner_len = inner_col.len();
        debug_assert!(inner_len <= len);
        builder.builder.append_column(inner_col);
        builder.validity.extend_constant(inner_len, true);
        // Avoid using `if branch`.
        let d = Scalar::default_value(typ);
        let defaults = ColumnBuilder::repeat(&d.as_ref(), len - inner_len, typ).build();
        builder.builder.append_column(&defaults);
        builder.validity.extend_constant(len - inner_len, false);
    }
}
```

以上便是 `Unnest` 函数执行的整个流程。

## 其他

### Table Function

`Unnest`还能作为 table function 存在。

```sql
select * from unnest([[1,2],[3]]);
----
1
2
3
```

### Unnest 多维（嵌套）数组

从上面的 SQL 例子中可以看出，Databend 对齐 PostgreSQL，会将整个嵌套数组全部展开。因此，在语义检查以及计划构建阶段，`Unnest` 表达式/计划的返回类型也应该全部展开。否则会在分布式等场景下出现 BUG（序列化与反序列化的 schema 不匹配）。

相关方法：

```Rust
impl Column {
    /// Unnest a nested column into one column.
    pub fn unnest(&self) -> Self {
        match self {
            Column::Array(array) => array.underlying_column().unnest(),
            col => col.clone(),
        }
    }
}

impl<T: ValueType> ArrayColumn<T> {
    pub fn underlying_column(&self) -> T::Column {
        debug_assert!(!self.offsets.is_empty());
        let range = *self.offsets.first().unwrap() as usize..*self.offsets.last().unwrap() as usize;
        T::slice_column(&self.values, range)
    }
}

impl DataType {
    pub fn unnest(&self) -> Self {
        match self {
            DataType::Array(ty) => ty.unnest(),
            _ => self.clone(),
        }
    }
}
```
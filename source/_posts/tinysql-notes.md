---
title: TinySQL 实现笔记
date: 2021-09-29 14:17:39
tags:
  - 个人项目
categories:
  - 编程开发
---

原项目：https://github.com/tidb-incubator/tinysql

我的实现：https://github.com/RinChanNOWWW/tinysql-impl

TInyKV 之外的另一个 PingCAP incubator 的项目，实现一个微型 TiDB，即 TinySQL。正好十月中旬要参加 OB 的比赛，做这个刚好可以了解了解数据库 SQL 层的一些东西，这些在 CMU 15-445 的实验中没怎么深入。虽然 TiDB 的存储层是 KV 的形式，但是总体上的思想应该都大同小异。与之前 TinyKV 一样，这个文章用来做个笔记。

<!-- more -->

## Project 1

将行记录与索引（实际上是 KV）中的 Key 解码，得到行以及索引的相关信息：table id，row id，index id，index col value 等。没什么好说的。就是代码骨架中有很多现成的函数，比如自己从切片中一个一个的取。

## Project 2

本节利用 yacc 来进行 SQL 语法的补充，此 Project 中补充的是 Join 的语法，针对测试用例中的 join, left join, right join, join on 编写 yacc 文件即可。由于学编译原理时接触过这些所以很快就能上手，而且不会的话还可以看 [TiDB parser 的源码](https://github.com/pingcap/parser/)不是（

## Project 3

本节实现了 SQL DDL 中删除一列的操作，具体来说是实现 F1 Schema 变更算法，具体参考[这篇文章](https://github.com/ngaut/builddatabase/blob/master/f1/schema-change.md)，这篇文章的例子是添加索引，这里需要实现删除列，所以状态的变更是反着的，应该为：

``` 
public --> write-only --> delete-only --> (reorg) --> absent
```

在 `public --> write-only` 阶段进行 `adjustColumnInfoInDropColumn`，因为这之后要被删除的 Col 已经不可被读了。

## Project 4

### Part 1

本部分主要是对 Cascades Planner 进行实现，主要是体验一下实现 `OnTransform` 与  `PredicatePushDown` 这两个方法。

#### OnTransform

这个方法是为了探索所有可能的 SQL plan，来对 Memo Group 这个树形结构进行变换，来寻找更优的执行方案。在这层逻辑中，我们只需要考虑具体变换的内容即可，不用考虑上层的各种判断条件。

- `PushSelDownAggregation`：这里要实现从 `sel->agg->x` 到  `agg->sel->x` 或者 `sel->agg->sel->x` 的变换。当 `sel` 中的条件都存在于 `agg` 中就可以完全下推到 `agg` 之后（前者），否则就是后者。具体体现是判断 `sel` 中每一个 `condition` 的列是否存在于 `agg` 的 `group by` 的列当中，并做下记录。存在的列可以下推到 `agg` 之下，剩余的保留在原位置。
- `MergeAggregationProjection`：这里要实现从 `agg->projection->x` 到 `agg->x` 的变换。也就是把映射合入 `agg` 中，只保留需要的列。我们只需要将 `agg` 中的 `group by` 的列与 `aggregation function` 的参数进行映射变换即可，然后消除 `projection` 这一层 Group。

#### PredicatePushDown

这个方法为实际的逻辑计划实现谓词下推。这里实现了 `LogicalAggregation` 的谓词下推。这里的实现有点像上面 `OnTransform` 两个部分的结合。先通过 `group by` 的列找到可以下推的 `condtion`，然后保留其中有用的列，最后将选出来的谓词下推即可。

### Part 2

#### Count-Min Sketch

实现 CM Sketch 算法，包括新增 value 以及查询 value。其中每一行的 Hash 算法（查看 TiDB 源码得知）为：

```go
h1, h2 := murmur3.Sum128(bytes)
for i := range c.table {
    j := (h1 + h2*uint64(i)) % uint64(c.width)  // hash
}
```

新增 value 较为简单，算出 Hash 之后再将 table 中对应位置的 count 增加即可。查询按照文章 https://pingcap.com/zh/blog/tidb-source-code-reading-12 的说法，TiDB 使用了 Count-Mean-Min Sketch 算法，引入了一个噪声值 `(N - CM[i, j]) / (w-1)` (N 为总数，w 为 table 宽度)，然后取所有行的值减去噪音之后的估计值的中位数作为最后的估计值。

#### Join Reorder

本节实现了用动态规划算法找出最优 join 顺序。整体流程并没有想象中的那么简单。下面总结一下算法流程：

1. 以邻接矩阵的方式建立 Join 图。
2. 记录下表示相等的边与不是表示相等的边。
3. 从一个没有访问的点开始进行广度优先遍历，得到一个连通的 Join 节点序列。
4. 对于这些节点的 cost，进行动态规划算法得出最少 cost 的 Join 方案。
5. 如果有还有没有访问节点则回到 3。
6. 将收集到的所有 Join 方案结合到一起作为结果。

本流程的复杂并不在于算法的实现，而在于各种信息的收集：

- 需要收集用于 Join 的相等边，即 `[]joinGroupEqEdge`，这个是用来存放 Join 中覆盖的边，在动态规划算法中即会使用。

- 不是表示相等条件的边，即 `[]joinGroupNonEqEdge`，这个用来存放与 Join 无关的边，需要保留这个信息传递给后续调用链。

#### Access Path Selection

本节实现 Skyline Prune 启发式算法来排除效果一定更差的路径，重点在于实现 `compareCandidates` 函数。根据注释来实现：

- 比较两个 candidate 的 col，覆盖范围更大的 candidate 更优。
- 比较两个 candidate 是否 match physical property，match 的那个更优。
- 比较两个 candidate 是否只需要扫面一次，只扫描一次的那个更优。

最后综合以上三种情况，如果 x 在所有方面都不弱于 y，且有一点优于 y，则选 x，反之亦然。

## Project 5

WIP

## 参考

- https://github.com/pingcap/parser (Project 2)
- https://github.com/ngaut/builddatabase/blob/master/f1/schema-change.md (Project 3)
- https://pingcap.com/zh/blog/tidb-cascades-planner (Project 4 Part 1)
- https://github.com/pingcap/tidb (Project 3, Project 4)
- https://pingcap.com/zh/blog/tidb-source-code-reading-12 （Project 4 Part 2）

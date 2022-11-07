---
title: 【WIP】Databend Sort 优化
date: 2022-11-05 17:23:42
tags:
    - 数据库
categories:
    - 编程开发
---

最近 Databend 针对数据排序进行了一波性能优化，在此记录一番。

<!-- more -->

排序操作在数据库中十分常见与重要。最常见的用法是将数据表按照某几个字段进行排序，或者找出数据表中最大或最小的几行（TopN）。

```sql
SELECT * FROM table ORDER BY co11, col2; -- normal sort
SELECT col FROM table ORDER BY col LIMIT n; -- top n
```

Databend 的 [RECLUSTER TABLE](https://databend.rs/doc/reference/sql/ddl/clusterkey/dml-recluster-table) 命令也与排序息息相关，此命令会将数据表按照 [CLUSTER KEY](https://databend.rs/doc/reference/sql/ddl/clusterkey/) 重新组织，并最终使得底层数据分部遵循 `CLUSTER KEY` 有序排列。

排序在数据库中的重要性不言而喻。在优化之前，Databend 的排序实现比较简单，性能表现不太理想，所以需要进行重新设计与优化。

## 历史实现

Databend 曾经的排序实现的十分简单——那就是不断地进行归并排序。

![old-pipeline](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/databend-sort-optimization/old-pipeline.png)

如图所示，Databend 会为每一个数据源分组建立一条 Partial Sort 流水线，然后通过 Resize 管道将多个流水线合一，然后再进行一次整体的归并排序得到最终的全序数据。

### 性能分析

![old-flame](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/databend-sort-optimization/old-flame.png)

很容易看出，这种流水线的瓶颈主要在于 MergeSort 阶段（火焰图也印证了这一点）。在实现中，MergeSort 会等待上游数据全部输入才会开始进行归并排序，因为只有这样才能保证数据最终的有序。然而，这种设计的一个最直接的影响就是：MergeSort 阻塞了流水线，下游算子需要等待所有数据块全部堆积在最后一个 MergeSort 并进行归并排序之后才能拿到数据。

```sql
SELECT AVG(col) FROM (SELECT col FROM table ORDER BY col LIMIT n);
```

设想这样一条 SQL，`AVG(col)` 需要整个 `(SELECT col FROM table ORDER BY col LIMIT n)` 执行完毕后才能开始计算，这显然效率是十分低下的。

>  题外话：为了缓解排序阻塞带来的影响，Databend 会将 `LIMIT` 下推到排序阶段，使得归并排序得到目标数量行之后便可以直接输出，这在一定程度上优化了 TopN 。

所以直观的看来，有两个优化方向：第一个是并行化归并排序（见最后一节），第二种是让 MergeSort “流起来“，目前的实现选择了后者。

## 优化一：流式 MergeSort

![new-pipeline](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/databend-sort-optimization/new-pipeline.png)

如图所示，这里主要的改造是将 Resize 和第二个 MergSort 合并为了一个 MultiMergeSort，而此次优化的重点便是这个 MultiMergeSort。这个算子的主要思想很显而易见，那就是使用堆来进行多路归并。以小根堆为例，若 N 路升序序列都至少有一个元素被 push 入了堆（按照顺序），此时堆顶元素则为全局最小元素。依据这个特性，我们可以在从上游拉取数据并向堆中 push 元素的同时，pop 出堆顶元素输出到下游，使数据”流动“起来，让下游算子不用等到整个序列排序完成后才能开始工作。

### 实现细节

#### DataBlock Cursor

排序是粒度是行，但是在向量化执行下，算子之间传递的数据自然不会是一行一行的数据，而是一个一个的数据块，一个数据块中包含有多行。所以这里需要引入一个新的数据结构 `Cursor` 来表示具体的某一行数据。

```rust
struct Cursor {
    pub input_index: usize, 	// Cursor 指向的数据是来自于哪个 input port
    // pub block_index: usize, 	// Cursor 指向的是哪一个 data block。可以省略，因为 heap 中每一个 block 只可能来自于不同的 input port。
    pub row_index: usize, 		// Cursor 指向的是哪一行
    // other fields
}
```

我们每次只用将 `Cursor` 推入堆中即可。`Cursor` 间的比较即为对应行排序键之间的比较。

#### Heap 操作

针对堆的操作是整个 MultiMegeSort 的核心，整体算法流程大概为：

1. 若堆中元素不少于输入数，执行下一步；否则结束此流程。
2. 弹出堆顶 `Cursor`，将 `Cursor` 所指向的行推入待输出队列。
3. 将 `Cursor` 指向下一行，若已遍历 block 中所有行，则标记此 input 可以拉取下一个 block 数据；否则，将递增后的 `Cursor` 放回堆中。
4. 若待输出队列累计已达要求（limit 或 block_size），则退出此流程准备输出；否则，返回第 1 步。

对于上述第 2~3 步，有一个可以优化的地方，如下图所示。

![heap](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/databend-sort-optimization/heap.png)

我们其实不用每次都将 `Cursor` 回堆，徒增一次 O(log(N)) 的操作。如果递增后的 `Cursor` 仍然比下一个堆顶元素小，那么我们可以继续将递增后的 `Cursor` 所指向的行放入输出队列。更加特殊的，如果当前 `Cursor` 所指向的 block 的最后一行元素都比下一个堆顶元素小，那么可以直接将从当前 `Cursor` 起此 block 中的所有数据都放入待输出队列。

通过实验证明，这个小细节会让查询快很多。对于一些比较特殊的语句甚至有 3 倍以上的提升，比如下面这个语句：

```sql
SELECT * FROM numbers(10000000) ORDER BY number;
SELECT * FROM numbers(10000000) ORDER BY number DESC;
```

因为 `numbers` 产生的 block 之间没有数据交叉，每个 block 之间本身就具有顺序。

这整体部分的代码如下。

```rust
while self.heap.len() >= nums_active_inputs && !need_output {
    match self.heap.pop() {
        Some(Reverse(mut cursor)) => {
            let input_index = cursor.input_index;
            let block_index = self.blocks[input_index].len() - 1;
            if self.heap.is_empty() {
                // If there is no other block in the heap, we can drain the whole block.
                while !cursor.is_finished() {
                    self.in_progess_rows
                    .push((input_index, block_index, cursor.advance()));
                    if let Some(limit) = self.limit {
                        if self.in_progess_rows.len() == limit {
                            need_output = true;
                            break;
                        }
                    }
                }
                // We have read all rows of this block, need to read a new one.
                self.cursor_finished[input_index] = true;
            } else {
                let next_cursor = &self.heap.peek().unwrap().0;
                while !cursor.is_finished() && cursor.lt(next_cursor) {
                    // If the cursor is smaller than the next cursor, don't need to push the cursor back to the heap.
                    self.in_progess_rows
                    .push((input_index, block_index, cursor.advance()));
                    if let Some(limit) = self.limit {
                        if self.in_progess_rows.len() == limit {
                            need_output = true;
                            break;
                        }
                    }
                }
                if !cursor.is_finished() {
                    self.heap.push(Reverse(cursor));
                } else {
                    // We have read all rows of this block, need to read a new one.
                    self.cursor_finished[input_index] = true;
                }
            }
            // Reach the block size, need to output.
            if self.in_progess_rows.len() >= self.block_size {
                need_output = true;
            }
        }
        None => {
            // Special case: self.heap.len() == 0 && nums_active_inputs == 0.
            // `self.in_progress_rows` cannot be empty.
            // If reach here, it means that all inputs are finished but `self.heap` is not empty before the while loop.
            // Therefore, when reach here, data in `self.heap` is all drained into `self.in_progress_rows`.
            debug_assert!(!self.in_progess_rows.is_empty());
            self.state = ProcessorState::Output;
            break;
        }
    }
}
```

#### Processor 状态机

由于 Databend 的流水线框架基于 Morsel-Driven Parallelism 的，所以需要为每一个 Processor 算子设计一个状态机。这里简单描述一下 MultiMergeSort 的状态机模型。整体状态机大致如下图所示，其中省略了一些边界情况。

![state-machine](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/databend-sort-optimization/state-machine.png)

- Consume: 拉取数据 block，转为 Preserve 状态。
- Preserve: 核心逻辑状态。将 block 推入堆中并执行上一节所示的堆操作流程。若可以进行输出，则转为 Output 状态；否则回到 Consume 状态进行下一波数据拉取。
- Output: 构造输出 block，转为 Generated 状态。
- Generated: 将需要输出的 block 推入 output port。

### 性能分析

![new-flame](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/databend-sort-optimization/new-flame.png)

上图是流化 MergeSort 之后（没有进行堆操作优化）的火焰图。可见，性能瓶颈从归并排序转换了堆操作，其中推操作的瓶颈又在于 `Cursor` 之间的比较运算。所以接下来便引出了本次的第二个优化点：Row Format。

## 优化二：Row Format

TODO

### 性能分析

TODO

## 其他优化点

1. 并行二路归并排序

DuckDB 使用这种方法来优化二路归并排序。并行二路归并排序的实现来源于论文 [Merge Path - A Visually Intuitive Approach to Parallel Merging](https://arxiv.org/pdf/1406.2628.pdf)，其基本思想是将找到两个有序序列中的排序交叉点，将两个序列按照交叉点分组，使得每个组内的两个数据部分独自归并排序，最后将所有块的结果合并。序列的分组使得每个组的归并排序可以并行执行，不会互相影响，最后直接这便可得到全序结果。

DuckDB 排序优化博客：https://duckdb.org/2021/08/27/external-sorting.html。

2. 基数排序

DuckDB 与 ClickHouse 等数据库都引入了基数排序。基数排序是一种[非比较的基于分布的排序方式](https://en.wikipedia.org/wiki/Sorting_algorithm#Non-comparison_sorts)，时间复杂度为 O(nk)，其中 k 是排序键的宽度（比如 Int32 的宽度是 4 字节），n 是排序键个数。当 n 很大的时候，基数排序的性能优势就会显现出来。

## 局限性

如果排序键为聚合函数的结果，以目前的实现来说，用于排序的始终就只有一条流水线，无法进行流化多路归并。需要针对这种情况优化流水线。

```sql
SELECT COUNT(*) as c FROM table ORDER BY c;
```
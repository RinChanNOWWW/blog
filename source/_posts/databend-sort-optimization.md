---
title: Databend Sort 优化
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

#### Processor 状态机

### 性能分析

## 优化二：ROW FORMAT

### 性能分析



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
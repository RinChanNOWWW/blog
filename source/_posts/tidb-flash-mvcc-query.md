---
title: TiDB Flashback 之 MVCC Query 的实现思路
date: 2021-12-30 18:59:05
tags:
    - Hackthon
    - 数据库
categories:
    - 编程开发
---

有幸和 [@disking](https://github.com/disksing) 与 [@JmPotato](https://github.com/JmPotato) 两位哥哥一同参加了今年（2021）TiDB 社区举办的 Hackthon。我们的项目简单来说就是基于 TiDB 的 MVCC 的特性实现一些新的功能。项目的 RFC 文档：https://github.com/Long-Live-the-DoDo/rfc 。

<!-- more -->

在本项目我负责第一个功能点：MVCC Query in SQL。简单来说就是为 TiDB 的数据表增加两个虚拟列 `_tidb_mvcc_ts` 与 `_tidb_mvcc_op`。众所周知，TiDB 中每一个数据表中的数据都是以 Key-Value 的形式存放在 TiKV 中，Key 是一条行记录的标识，它带有 MVCC 版本信息（ts）；而 Value 就是这一条行记录的具体内容，它多列的数据拼接而成。TiDB 在读取数据时只会读取当前事务下能看到的最新数据，而我要做的就是能让它看到所有版本的数据。

一言以蔽之，我要实现的便是当 SQL 语句中存在 `_tidb_mvcc_ts` 或 `_tidb_mvcc_op` 两列时，会查询到所有的历史版本（以及它们的 MVCC 时间戳与操作类型）。

## 准备工作

除了 TiDB 与 TiKV 的开发环境准备之外，需要做的一个准备工作就是了解 TiDB 和 TiKV 的代码结构与它们的数据流，也就是要去大致了解它们的源码，而这也是最耗时间的一个过程，所以我的代码量并不大，但是却花了很长时间才写完。

于是我根据我需要改动的部分，结合 PingCAP 的官方博客，对源码进行了一波学习：

- Select 流程：
    - [TiDB 源码阅读系列文章（三）SQL 的一生](https://pingcap.com/zh/blog/tidb-source-code-reading-3)   
    - [TiDB 源码阅读系列文章（六）Select 语句概览](https://pingcap.com/zh/blog/tidb-source-code-reading-6)
- 如何将查询下推到 TiKV 并执行：
    - [TiKV 源码解析系列文章（十四）Coprocessor 概览](https://pingcap.com/zh/blog/tikv-source-code-reading-14)
    - [TiKV 源码解析系列文章（十六）TiKV Coprocessor Executor 源码解析](https://pingcap.com/zh/blog/tikv-source-code-reading-16)
- Insert 流程：
    - [TiDB 源码阅读系列文章（四）Insert 语句概览](https://pingcap.com/zh/blog/tidb-source-code-reading-4)
    - [TiDB 源码阅读系列文章（十六）INSERT 语句详解](https://pingcap.com/zh/blog/tidb-source-code-reading-16)
- 一条 SQL 语句的具体执行流程：
    - [TiDB 源码阅读系列文章（二十三）Prepare/Execute 请求处理](https://pingcap.com/zh/blog/tidb-source-code-reading-23)
- TiKV MVCC 读写流程：
    - [TiKV 源码解析系列文章（十三）MVCC 数据读取](https://pingcap.com/zh/blog/tikv-source-code-reading-13)

## TiDB 侧实现

TiDB 负责 SQL 语句的解析与查询计划的构建。所以 TiDB 侧的修改主要分为两个部分：第一个是查询计划构建引入虚拟列，第二个是提示 TiKV 需要读取所有版本的数据。下图是 TiDB 中需要修改的部分：

![mvcc_query_in_tidb](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/tidb-flash-mvcc-query/mvcc_query_in_tidb.png)

### 查询计划构建引入虚拟列

以 `select _tidb_mvcc_ts from t` 为例，当 `PlanBuilder` 读取到 `t` 时，它会调用 [buildDataSource](https://github.com/Long-Live-the-DoDo/tidb/blob/3e4bc47d0d247ec81a09ed9eb6bb7c3e3137797c/planner/core/logical_plan_builder.go#L3953) 为查询计划构造逻辑上的数据源（也就是要查询的表），如果这里不做任何处理，当 TiDB 执行前进行各种元信息（比如查询的列是否存在等）时就会认为 `_tidb_mvcc_ts` 不是表 `t` 的列，从而执行失败，所以我们只需要在 `buildDataSource` 的时候加入虚拟列即可，让 TiDB 认为这个表就是有 `_tidb_mvcc_ts` 和 `_tidb_mvcc_op` 这两列的：

```go
// planner/core/logical_plan_builder.go

func (b *PlanBuilder) buildDataSource(ctx context.Context, tn *ast.TableName, asName *model.CIStr) (LogicalPlan, error) {
    // ...
    // add _tidb_mvcc_ts to columns.
	mvccTsCol := ds.newExtraMVCCTsSchemaCol()
	ds.Columns = append(ds.Columns, model.NewExtraMVCCTsColInfo())
	schema.Append(mvccTsCol)
	names = append(names, &types.FieldName{
		DBName:      dbName,
		TblName:     tableInfo.Name,
		ColName:     model.ExtraMVCCTsName,
		OrigColName: model.ExtraMVCCTsName,
	})
	ds.TblCols = append(ds.TblCols, mvccTsCol)
	// add _tidb_mvcc_op to columns.
	mvccOpCol := ds.newExtraMVCCOpSchemaCol()
	ds.Columns = append(ds.Columns, model.NewExtraMVCCOpColInfo())
	schema.Append(mvccOpCol)
	names = append(names, &types.FieldName{
		DBName:      dbName,
		TblName:     tableInfo.Name,
		ColName:     model.ExtraMVCCOpName,
		OrigColName: model.ExtraMVCCOpName,
	})
	ds.TblCols = append(ds.TblCols, mvccOpCol)
    // ...
}
```

其中与 MVCC 有关的数据结构与函数都是本次新加入的。

### NeedMvcc 标志

我在查询计划构建的过程中引入了 `NeedMvcc` 标志来标记本次查询是否需要 MVCC 信息。具体来说，是在遍历语法树生成查询计划时，如果遇到了 `_tidb_mvcc_ts` 或 `_tidb_mvcc_op`，则将这个标识即为 `true`。这里的实现比较简单粗暴，直接将 `NeedMvcc` 存到了 `SessionCtx` 中，也就是存到了整个 `Session` 的上下文中（所以用完之后需要清理为 `false`）。我觉得更优雅的方式是存放到 `LogicPlan` 与 `PhysicPlan` 中，然后层层传递到 `Executor`，但为了简单快捷的实现，我就直接放到了会话上下文中。

设置 `NeedMvcc` 具体发生在 `exoressionRewriter` 这个 `Visitor` 执行 [Leave](https://github.com/Long-Live-the-DoDo/tidb/blob/3e4bc47d0d247ec81a09ed9eb6bb7c3e3137797c/planner/core/expression_rewriter.go#L1042) 时，如果它需要接受的原本的语法树节点是 `ast.ColumnNameExpr` 类型，也就是列名时，我们可以检查这个列名是否是 `_tidb_mvcc_ts` 或 `_tidb_mvcc_op`，如果是，则将 `NeedMvcc` 设置为 `true`，否则不变：

```go
// planner/core/expression_rewriter.go

// Leave implements Visitor interface.
func (er *expressionRewriter) Leave(originInNode ast.Node) (retNode ast.Node, ok bool) {
    // ...
    case v := inNode.(type) {
        // ...
        case *ast.ColumnNameExpr:
            if v.Name.Name.L == model.ExtraMVCCOpName.L || v.Name.Name.L == model.ExtraMVCCTsName.L {
                er.sctx.GetSessionVars().NeedMvcc = true
            }
        // ...
    }
    // ...
}
```

接下来就是在通过物理计划构造 `Executor` 并构造对 TiKV 的 RPC 请求时，加入 `NeedMVCC` 这个标识字段即可：

```go
// planner/core/plan_to_pb.go

// ToPB implements PhysicalPlan ToPB interface.
func (p *PhysicalTableScan) ToPB(ctx sessionctx.Context, storeType kv.StoreType) (*tipb.Executor, error) {
	tsExec := tables.BuildTableScanFromInfos(p.Table, p.Columns, ctx.GetSessionVars().NeedMvcc)
	// clear the need mvcc flag
	ctx.GetSessionVars().NeedMvcc = false
	// ...
}

// table/tables/tables.go

// BuildTableScanFromInfos build tipb.TableScan with *model.TableInfo and *model.ColumnInfo.
func BuildTableScanFromInfos(tableInfo *model.TableInfo, columnInfos []*model.ColumnInfo, needMvcc bool) *tipb.TableScan {
	pkColIds := TryGetCommonPkColumnIds(tableInfo)
	tsExec := &tipb.TableScan{
		TableId:          tableInfo.ID,
		Columns:          util.ColumnsToProto(columnInfos, tableInfo.PKIsHandle),
		PrimaryColumnIds: pkColIds,
		NeedMvcc:         needMvcc,
	}
    // ...
}
```
接下来就可以交给 TiKV 来管了。对于 SQL 语句中的投影，选择等操作，无需修改，本身是兼容的。只要我们能够拿到想要的数据，构造好了想要的数据表 Schema，就可以实现剩余的功能。

### 其他

由于在后续 TiKV 读取行数据写入 MVCC 信息的实现中（见下文），采用了原始的简单编码方式（colID1 typed_value1 colID2 typed_value...），所以需要统一 TiDB 和 TiKV 的存储数据编解码方式。为了简单起见，我直接抛弃了 TiDB 新引入的编码方式：

```go
// tablecodec/tablecodec.go

// EncodeRow encode row data and column ids into a slice of byte.
// valBuf and values pass by caller, for reducing EncodeRow allocates temporary bufs. If you pass valBuf and values as nil,
// EncodeRow will allocate it.
func EncodeRow(sc *stmtctx.StatementContext, row []types.Datum, colIDs []int64, valBuf []byte, values []types.Datum, e *rowcodec.Encoder) ([]byte, error) {
	if len(row) != len(colIDs) {
		return nil, errors.Errorf("EncodeRow error: data and columnID count not match %d vs %d", len(row), len(colIDs))
	}
	// For hackthon: disable the new encoding mode.
	// if e.Enable {
	// 	return e.Encode(sc, colIDs, row, valBuf)
	// }
	return EncodeOldRow(sc, row, colIDs, valBuf, values)
}
```

TiDB 侧所有修改见 https://github.com/Long-Live-the-DoDo/tidb/pull/1 。

## TiKV 侧实现

通过阅读 Coprocessor 的相关源码可以得知，整个 `select` 语句的执行过程可以大致拆分为：

![mvcc_query_in_tikv](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/tidb-flash-mvcc-query/mvcc_query_in_tikv.png)

### 构造 executor 及其中的 scanner

1. 首先通过通过 [build_executors](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/components/tidb_query_executors/src/runner.rs#L153) 构造 `executor`。由于 `select` 语句对应的是 `ExecType::TypeTableScan`，所以相应的 `executor` 会通过 [BatchTableScanExecutor::new](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/components/tidb_query_executors/src/table_scan_executor.rs#L36) 生成。而 `build_executors` 这里可以获取到 TiDB 发来的 RPC 请求，我们便可以从请求中提取出 `NeedMvcc` 字段传给 `BatchTableScanExecutor` 以便后续使用。
2. 在 `BatchTableScanExecutor` 中，又会调用 [ScanExecutor::new](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/components/tidb_query_executors/src/util/scan_executor.rs#L63) 生成一个 scanner wrapper。于是再将依次将 `NeedMvcc` 传递给 `ScanExecutor` 这个 wrapper。
3. 由于 `ScanExecutor` 只是个 wrapper，所以它还会将内部的实际 scanner 定义为 [RangeScanner](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/components/tidb_query_common/src/storage/scanner.rs#L42)。于是再将 `NeedMvcc` 传递给 `RangeScanner` 保存起来。当之后 `RangeScanner` 执行 scan 逻辑时便可以获取到 `NeedMvcc` 了。
4. 第一次 scan 时，`RangeScanner` 会通过会调用内部成员 `storage` 的 `begin_scan` 接口。而 `RangeScanner` 中的 `storage` 实际上是 [TiKVStorage](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/coprocessor/dag/storage_impl.rs#L14)。
5. 在 [TiKVStorage::begin_scan](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/coprocessor/dag/storage_impl.rs#L39) 中，又会根据 `TiKVSttorage<S: Store>` 的内部成员 `store: S` 的类型生成其对应的 scanner。这里的 `store` 其实是 [SnapshotStore](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/storage/txn/store.rs#L261)。`TiKVStorage` 会通过 [SnapshotStore::scanner](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/storage/txn/store.rs#L354) 生成最终访问物理存储的 scanner。于是 `NeedMvcc` 就可以通过 `begin_scan` 函数调用传递给 `SnapshotStore::scanner` 构造相应的 scanner 了。
6. `SnapshotStore::scanner` 通过 [ScannerBuilder](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/storage/mvcc/reader/scanner/mod.rs#L26) 按照需求构造相应的 scanner。在原本的实现了两种 scanner：`Scanner::Forward` 与 `Scanner::Backward`，分别用于顺序和逆序扫描，并提供 `next` 接口返回每一次迭代读到的 Key-Value 值。于是我就知道了，我这里需要新增一个 `Scanner::ForwardWithMvcc` 来用于在顺序扫描中返回所有版本的 Key-Value。

我找到这条调用链其实是个逆序的过程，先找到最基本的 scanner，再向上溯源找到最基本的调用者。

### scanner 迭代获取数据

1. 构造完 executor 后就可以开始执行了。对于 `select` 也就是 `TableScan` 来说，对于每一行数据的读取实际上就是不断调用 [ForwardScanner::read_next](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/storage/mvcc/reader/scanner/forward.rs#L168) 这个方法（拿顺序遍历举例）。
2. `read_next` 方法会按照 Key 遍历 `write` 列族拿到 write 的值，然后再通过 `handle_write` 方法执行迭代器读取数据的实际逻辑。所以说，我们最终的目的就是为 `Scanner::ForwardWithMvcc` 实现它的 `handle_write` 方法。
3. `handle_write` 的执行逻辑因 `Scanner` 采用的 [ScanPolicy](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/storage/mvcc/reader/scanner/forward.rs#L18) 而异。我们要实现的 [WithMvccInfoPolicy](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/storage/mvcc/reader/scanner/forward.rs#L487) 和原本的 `Scanner::Forward` 使用的 [LatestKvPolicy](https://github.com/Long-Live-the-DoDo/tikv/blob/27ebbf45b7918e3674b298afa835b027dafcd5cb/src/storage/mvcc/reader/scanner/forward.rs#L365) 基本完全一致。`LatestKvPolicy` 遇到某 user key 的最新版本 value 时，会返回这个 value，并通过调用

```rust
// src/storage/mvcc/reader/scanner/forward.rs

impl<S: Snapshot> ScanPolicy<S> for LatestKvPolicy {
	fn handle_write(
        &mut self,
        current_user_key: Key,
        _: TimeStamp,
        cfg: &mut ScannerConfig<S>,
        cursors: &mut Cursors<S>,
        statistics: &mut Statistics,
    ) -> Result<HandleRes<Self::Output>> {
		// find the next value
		// ...
		cursors.move_write_cursor_to_next_user_key(&current_user_key, statistics)?;
		// return the value
		Ok(match value {
            Some(v) => HandleRes::Return((current_user_key, v)),
            _ => HandleRes::Skip(current_user_key),
        })
	} 
}
```
跳过这个 user key 之下所有之前的老版本。在 `WithMvccInfoPolicy` 下，我们并不采用这个逻辑，而是继续往下读。

```rust
// src/storage/mvcc/reader/scanner/forward.rs

impl<S: Snapshot> ScanPolicy<S> for WithMvccInfoPolicy {
	fn handle_write(
        &mut self,
        current_user_key: Key,
        commit_ts: TimeStamp,
        cfg: &mut ScannerConfig<S>,
        cursors: &mut Cursors<S>,
        statistics: &mut Statistics,
    ) -> Result<HandleRes<Self::Output>> {
		// find the next value
		// ...
		// do not move write cursor to next user key,
		// but move to next key.
		cursors.write.next(&mut statistics.write);
		// add mvcc info and return the value
		Ok(match value {
            Some(mut v) => {
                self.append_row_with_mvcc_info(&mut v, commit_ts, write_type)?;
                HandleRes::Return((current_user_key, v))
            }
            _ => {
                if write_type == WriteType::Delete {
                    let mut v = vec![];
                    self.append_row_with_mvcc_info(&mut v, commit_ts, write_type)?;
                    HandleRes::Return((current_user_key, v))
                } else {
                    HandleRes::Skip(current_user_key)
                }
            }
        })
	}
}
```
`WithMvccInfoPolicy` 还有一点不同的是，遇到 `Delete` 的时，它也会返回一个 value。

4. `WithMvccInfoPolicy` 中的 `handle_write` 在返回 value 之前，还需要为 value 中添加入 `_tidb_mvcc_ts` 与 `_tidb_mvcc_op` 这两个虚拟列的数据。就像上面 TiDB 侧的实现中所说，这里采用了一种简单粗暴的编码方式，直接将 [colID, typed_value] 依次编码进最后的 value 字节数组中。

```rust
impl WithMvccInfoPolicy {
    fn append_row_with_mvcc_info(
        &self,
        value: &mut Value,
        commit_ts: TimeStamp,
        write_type: WriteType,
    ) -> codec::Result<()> {
        // append mvcc_ts_col_id
        codec::number::NumberEncoder::write_u8(value, VAR_INT_FLAG)?;
        codec::number::NumberEncoder::write_var_i64(value, EXTRA_MVCC_TS_COL_ID)?;
        // append mvcc_ts
        codec::number::NumberEncoder::write_u8(value, VAR_UINT_FLAG)?;
        codec::number::NumberEncoder::write_var_u64(value, commit_ts.physical())?;
        // append mvcc_op_col_id
        codec::number::NumberEncoder::write_u8(value, VAR_INT_FLAG)?;
        codec::number::NumberEncoder::write_var_i64(value, EXTRA_MVCC_OP_COL_ID)?;
        // append mvcc_op
        codec::number::NumberEncoder::write_u8(value, COMPACT_BYTES_FLAG)?;
        codec::byte::CompactByteEncoder::write_compact_bytes(
            value,
            &write_type.to_string().as_bytes().to_vec(),
        )?;
        value.shrink_to_fit();
        std::result::Result::Ok(())
    }
}
```

TiKV 侧所有修改见 https://github.com/Long-Live-the-DoDo/tikv/pull/1 。

## 执行效果

- select

![select](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/tidb-flash-mvcc-query/select.png)

- update

![update](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/tidb-flash-mvcc-query/update.png)

- delete

![delete](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/tidb-flash-mvcc-query/delete.png)

- delete 后 update

![update_after_delete](https://cdn.jsdelivr.net/gh/RinChanNOWWW/jsdelivrp-cdn@master/blog/images/tidb-flash-mvcc-query/update_after_delete.png)

## 还可以继续完善的点

- 将 `NeedMvcc` 存放在 `SessionContextVars` 中并不优雅，最好放入查询计划的相应结构中。
- `delete from t where _tidb_mvcc_ts=xxx` 这样的语句存在二义性，后续可能会做成删除对应的 MVCC 记录。
- 没有做 `_tidb_mvcc_ts` 与 `_tidb_mvcc_op` 相关的限制。例如，创建表时不允许以这两个名字命名、不允许修改这两列的值等。也可以做成，当修改这两列时直接进行 KV 更新操作（不过这种感觉没什么意义）。
- 在 TiKV 中只实现了 MVCC 的 `ForwardScan` 没有实现 `BackwardScan`，这可能造成某些使用了 `BackwardScan` 的语句的不兼容。
- 加入虚拟列的编码方式过于简单粗暴，可以考虑对其现阶段的 TiDB 适配所有编码方式。
- ……
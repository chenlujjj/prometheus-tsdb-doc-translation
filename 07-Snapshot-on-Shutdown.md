# Prometheus TSDB (Part 7): Snapshot on Shutdown

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-snapshot-on-shutdown/

## 简介

在第2篇文章中，我们看到 TSDB 使用预写式日志（WAL）来提供应对崩溃的持久性。但这也会使得 Prometheus 在达到一定使用规模时重启变慢，因为回放 Checkpoint+WAL 需要时间。

本文中我们将理解一项在 Prometheus v2.30.0 引入的新特性：在关闭时对内存中的数据做快照，这样就可以跳过WAL回放来更快地重启。

## 关于快照

TSDB 里的快照指的是在某个特定时间，对 TSDB 的内存中数据的一个只读并且静态的视图。

快照中包含了如下（按顺序）：

1. Head block 中所有的时序和每个时序的内存中 chunk。（回忆下第3篇文章：除了最后一个 chunk，其他都在磁盘上并且做了内存映射）。

2. Head block 中所有的墓碑。

3. Head block 中所有的标本（exemplars）。

受第2篇文章中的 checkpoints 的启发，我们将这些快照命名为 `chunk_snapshot.X.Y`，其中 `X` 是做快照时的最后一个 WAL 分段数，而 `Y` 是数据写入 `X` WAL 分段的字节偏移量。

```
data
├── 01EM6Q6A1YPX4G9TEB20J22B2R
|   ├── chunks
|   |   ├── 000001
|   |   └── 000002
|   ├── index
|   ├── meta.json
|   └── tombstones
├── chunk_snapshot.000005.12345
|   ├── 000001
|   └── 000002
├── chunks_head
|   ├── 000001
|   └── 000002
└── wal
   ├── checkpoint.000003
   |   ├── 000000
   |   └── 000001
   ├── 000004
   └── 000005
```

我们在停止所有的写入后关闭 TSDB 时制作快照。如果 TSDB 时突然停止的，那么不会制作新的快照，上一次优雅关闭的快照保留下来。

这个特性默认没有启用，可以使用 `--enable-feature=memory-snapshot-on-shutdown` 来开启。

## 快照格式

快照使用 [Prometheus TSDB 一般性的 WAL 实现](https://ganeshvernekar.com/blog/prometheus-tsdb-snapshot-on-shutdown/)，定义了3种新的记录格式。

快照中记录的顺序总是：

1. 序列记录（>=0）：这个记录是对单个序列的快照。每个序列会写一条记录，以无序的形式。它包含了序列的元数据和内存中的 chunk 数据（如果存在的话）。

2. 墓碑记录（0-1）：所有序列的记录完成后，我们写一条包含了 Head block 中所有墓碑的墓碑记录。一条包含了所有墓碑的记录会被写入到快照中。

3. 标本记录（>=0）：最后，当批量处理每个记录中的标本时，我们写入一个或多个标本记录。标本按照它们被写入循环缓冲区的顺序排列。

这些记录的格式可以在[这里](https://github.com/prometheus/prometheus/blob/main/tsdb/docs/format/memory_snapshot.md)找到，我们不会在本文中讨论。

## 恢复内存中状态

有了 `chunk_snapshot.X.Y`，我们可以忽略在第 `X` 分段的 `Y` 偏移量之前的 WAL 而只要回放这之后的 WAL，因为快照和内存映射的 chunks 代表了直到这个点的 WAL 的回放状态。

所以当快照开启时，回放数据来恢复 Head 的过程如下：

1. 按照在第3篇文章中描述的那样迭代所有的内存映射的 chunks，构建 map

2. 从最新的 `chunk_snapshot.X.Y` 的逐个迭代序列记录。对于每条序列记录，重新创建在内存中的带有标签的序列和记录中的内存中的 chunk。

和处理 WAL 中的 `Series` 记录类似，我们寻找对应于这个序列引用的内存映射 chunk时，然后将它和这个序列连在一起。

3. 如果有的话，从快照中读取墓碑记录，然后将其恢复到内存中。

4. 如果有的话，逐个地迭代标本记录，将其按照相同的顺序放回到循环缓冲区里。

5. 回放完内存映射的 chunks 和快照后，我们继续回放从第 `X` 分段的 `Y` 字节偏移量的WAL。如果有标数 `>=X` 的 WAL checkpoints，我们会在回放 WAL 之前回放最后一个 checkpoint。

在大多数情况下（即优雅关闭），不会有需要回放的 WAL，这是因为快照是在关闭期间停止写入后制作的。如果 Prometheus 碰巧突然崩溃或者关闭了，就会有需要回放的 WAL 和 Checkpoint。

## 更快的重启

当我们谈论重启时，不仅要考虑回放磁盘上的数据来恢复内存状态的时间，也要考虑关闭消耗的时间，因为制作快照会给关闭增加一些延迟。

写入快照消耗的时间是在秒的数量级，对于1百万序列通常低于1分钟。回放 checkpoint 消耗的时间也是在秒级别。然而对于同样数量的序列，回放 WAL 可以消耗几分钟。

通过在优雅重启时跳过 WAL 回放，我们观察到*重启*时间减少了 50～80%。

## 注意事项

* 快照在启用时会占用额外的磁盘空间，而且不会替代已经存在的快照。

* 关闭会消耗一些时间，这取决于有多少序列和磁盘的写入速度。因此，要对 Prometheus pod 设置好 [pod 优雅终止时期](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)（或者类似选项）。

## 问题解答

*为什么只在关闭时制作快照？*

当我们查看一个 sample 写入磁盘（或者是在压缩时重写）的次数时，这并不太多。如果我们在 Prometheus 运行时间隔地制作快照的话，这会大幅度增加一个 sample 写入磁盘的次数，由此导致不必要的**写放大**。所以我们选择了处理大多数情况下发生的优雅关闭，而崩溃时将读取部分 WAL，这取决于磁盘上的最后一个快照。

*为什么还需要 WAL？*

如果 Prometheus 碰巧由于种种原因崩溃了，因为没有快照，为了持久性我们就需要 WAL。另外，[远程写](https://prometheus.io/docs/practices/remote_write/)也依赖于 WAL。


## 参考代码

制作和读取快照的代码在 [`tsdb/head_wal.go`](https://github.com/prometheus/prometheus/blob/main/tsdb/head_wal.go)中。

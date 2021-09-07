# Prometheus TSDB (Part 2): WAL and Checkpoint

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-wal-and-checkpoint/


## 简介

在 TSDB 系列博客的第1篇中提到，我们首先将到来的 samples 写入 [Write-Ahead-Log (WAL)](https://en.wikipedia.org/wiki/Write-ahead_logging) 中，以保证其持久性，当WAL被截断时，将创建一个 checkpoint。本文我们将简要讨论 WAL 的基础知识，然后深入探讨在Prometheus 的 TSDB 中 WAL 和 checkpoints 是如何设计的。

## WAL 基础

WAL 是数据库中事件发生的*顺序*日志。在写/修改/删除数据库中的数据**之前**，先将事件记录（追加）到 WAL 中，然后在数据库中执行必要的操作。

无论出于什么原因，如果机器或程序崩溃了，你可以使用 WAL 中记录的事件，将它们以同样的顺序回放来恢复数据。这对于内存数据库特别有用。如果没有 WAL，数据库一旦崩溃，内存中的所有数据都会丢失。

WAL 在关系数据库中被广泛使用，以提供数据库的持久性（来自ACID的D）。类似地，Prometheus 有 WAL 来为 Head block 提供持久性。Prometheus还使用 WAL 进行优雅重启，以恢复内存状态。

在 Prometheus 的上下文中，WAL 只用于记录事件和在启动时恢复内存状态。它不参与任何其他方式的读或写操作。

## 写入在 Prometheus TSDB 的 WAL

### 记录的类型

TSDB 中的写请求由 series 的标签值及其对应的 samples 组成。这给了我们两种类型的记录，`Series`和`Samples`。

`Series` 记录由写请求中所有 Series 的标签值组成。series 的创建会产生一个唯一的引用，可用于查找 series。因此，`Samples` 记录包含了写请求中对应 series 的引用和属于该 series 的 samples 列表。

最后一种类型的记录是用于删除请求的 `Tombstones`。它包含已删除的 series 引用以及要删除的时间范围。

这些记录的格式可以在[这里](https://github.com/prometheus/prometheus/blob/master/tsdb/docs/format/wal.md)找到，我们不会在本文中讨论。

### 写

所有包含 sample 的写请求，都会写入 `Samples` 记录。而对于每个 series，只会在第一次被看到时写入一次 `Series` 记录(也就是在 Head 中“创建”该series时)。

如果写请求包含一个新的 series，那么 `Series` 记录总是写在  `Samples` 记录之前，否则如果 `Samples` 记录在 `Series` 之前，那么在重放期间，`Samples`记录中的 series 引用将不会指向任何 series。

<!-- Series记录是在在Head中创建系列之后写入的，以在记录中存储引用，而Samples记录是在向Head添加示例之前写入的。

通过将所有不同的时间序列(和不同时间序列的样本)分组到同一个记录中，每个写请求只写一个序列和样本记录。如果请求中所有样品的系列已经存在于Head中，则只向WAL中写入一个sample记录。

当我们收到一个删除请求时，我们不会立即从内存中删除它。我们存储一些叫做“墓碑”的东西，它表示删除的序列和删除的时间范围。在处理删除请求之前，我们向WAL中写入一个Tombstones记录。 -->

### 在磁盘上长什么样

WAL 是用一系列数字编号的文件来存储的，每个文件默认 128MiB 大小。一个 WAL 文件被称作是一个“segment”。

```
data
└── wal
    ├── 000000
    ├── 000001
    └── 000002
```

文件大小有限制，使得对旧文件的垃圾回收更简单。正如你所想到的那样，序列号*总是*增大的。

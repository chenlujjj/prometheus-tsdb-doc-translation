# Prometheus TSDB (Part 2): WAL and Checkpoint

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-wal-and-checkpoint/


## 简介

在 TSDB 系列博客的第1篇中曾提到，我们首先将到来的 samples 写入 [预写式日志 (WAL)](https://en.wikipedia.org/wiki/Write-ahead_logging) 中，以保证其持久性，当WAL被截断时，将创建一个检查点。本文我们将简要讨论 WAL 的基础知识，然后深入探讨在 Prometheus 的 TSDB 中 WAL 和检查点是如何设计的。

## WAL 基础

WAL 是数据库中发生的事件的**连续**日志。在写入/修改/删除数据库中的数据**之前**，先将事件记录（追加）到 WAL 中，然后在数据库中执行必要的操作。

无论出于什么原因，如果机器或程序崩溃了，你可以使用 WAL 中记录的事件，将它们以同样的顺序回放来恢复数据。这对于内存数据库特别有用。如果没有 WAL，数据库一旦崩溃，内存中的所有数据都会丢失。

WAL 在关系数据库中被广泛使用，以提供数据库的[持久性(durability)](https://en.wikipedia.org/wiki/Durability_(database_systems))（来自 [ACID](https://en.wikipedia.org/wiki/ACID) 中的D）。类似地，Prometheus 有 WAL 来为 Head block 提供持久性。Prometheus 还使用 WAL 进行优雅重启，以恢复内存中的状态。

在 Prometheus 的上下文中，WAL 只用于记录事件和在启动时恢复内存中的状态。它不会以任何其他方式参与读或写操作。

## 写入在 Prometheus TSDB 的 WAL

### 记录的类型

TSDB 中的写请求由[序列](https://prometheus.io/docs/concepts/data_model/)的标签值及其对应的[samples](https://prometheus.io/docs/concepts/data_model/#samples) 组成。因此有了两种类型的记录：`Series`和`Samples`。

`Series` 记录由写请求中所有序列的标签值组成。序列的创建会产生一个唯一的引用，可用于查找序列。因此，`Samples` 记录中包含了对应序列的引用和写请求中属于该序列的 samples 列表。

最后一种类型的记录是用于删除请求的 `Tombstones`。它包含删除的序列的引用以及要删除的时间范围。

这些记录的格式可以在[这里](https://github.com/prometheus/prometheus/blob/master/tsdb/docs/format/wal.md)找到，我们不会在本文中讨论。

### 写入

所有包含 sample 的写请求，都会写入 `Samples` 记录。而对于每个序列，只会在第一次被出现时写入一次 `Series` 记录(也就是在 Head 中“创建”该序列时)。

如果写请求包含一个新的序列，那么 `Series` 记录总是写在  `Samples` 记录之前，否则如果 `Samples` 记录在 `Series` 之前，那么在回放时，`Samples`记录中的序列引用将无法指向任何序列。

`Series` 记录是在 Head 中创建序列*之后*写入的，以在记录中存储（序列的）引用，而 `Samples` 记录是在向 Head 添加 samples 之前写入的。

通过将同一个记录中所有不同的时间序列（和不同时间序列的 samples）聚合，每个写请求只写一个 `Series` 和 `Samples` 记录。如果请求中所有 samples
 的序列已经存在于 Head 中，那么只会把一个 `Samples` 记录写入到 WAL 中。

当我们收到一个删除请求时，我们不会立即从内存中删除它。我们存储一个叫“墓碑”的东西，它表示删除的序列和时间范围。在处理删除请求之前，我们向 WAL 中写入一个 `Tombstones` 记录。

### 它在磁盘上长什么样

WAL 是用一系列数字编号的文件来存储的，**每个文件默认 128MiB 大小**。一个 WAL 文件被称作是一个“**段(segment)**”。

```
data
└── wal
    ├── 000000
    ├── 000001
    └── 000002
```

文件大小的限制，使得对旧文件的垃圾回收更简单。正如你所想到的那样，序列号**总是增大**的。

## WAL 截断和创建检查点

我们需要定期地删除旧的 WAL 段，否则磁盘最终会占满，而且 TSDB 的启动会花费很长时间，因为它必须回放 WAL 中所有的事件（其中大部分会被丢弃，因为已经旧了）。一般地说，任何不再需要的数据，都要删除掉。

### WAL 截断

WAL 截断是在 Head block 截断**后**完成的（关于 Head 的截断参见第一篇文章）。文件不能被随机地删除，要删除的是前N个文件，不能在文件编号顺序中造成缺口。

由于写请求可能是随机的，所以不通过遍历所有的记录来判断一个 WAL 段中的 samples 的时间范围并不是一件容易或者有效率的事情。因此我们会删除前2/3的段。

```
data
└── wal
    ├── 000000
    ├── 000001
    ├── 000002
    ├── 000003
    ├── 000004
    └── 000005
```

在上面的例子中，文件 `000000`, `000001`, `000002`, `000003` 会被删除。


这里有件事情要注意：序列记录只会被写一次，所以如果盲目地删除 WAL 段，就会丢失这些记录，因而无法在启动时恢复这些序列。另一方面，在前2/3的段中可能有还没从 Head 中截断的 samples，这些 samples 也会丢失。这也就是为什么需要检查点。

### 创建检查点

在截断 WAL **之前**，我们从待删除的 WAL 段中创建一个 “检查点”。你可以把一个检查点当作是一个过滤后的 WAL。考虑下如果 Head 的截断是对于时间 `T` 以前的数据，以上面的 WAL 文件布局为例，创建检查点的操作会按顺序遍历在 `000000`, `000001`, `000002`, `000003` 中的所有记录，并且：

1. 丢弃所有不在 Head 中的序列对应的序列记录。

2. 丢弃所有在时间 `T` 以前的 samples。

3. 丢弃所有在时间 `T` 以前的墓碑。

4. 保留剩下的序列、samples 和墓碑记录（以它们在WAL 中出现的顺序保留）。

丢弃操作在丢弃记录中一些不必要的项时（因为一个记录可能包含多个序列，sample或者墓碑）也是一个重写操作。

这样就不会丢失那些依然在 Head 中的序列，samples和墓碑了。检查点被命名为 `checkpoint.X`，其中 `X` 是在检查点创建时的最后一个段的编号（在这个例子中是 `000003`；你将会在下一部分知道为什么这么命名）。

在 WAL 截断和创建检查点后，磁盘上的文件是这个样子（检查点就像是另一个 WAL）：

```
data
└── wal
    ├── checkpoint.000003
    |   ├── 000000
    |   └── 000001
    ├── 000004
    └── 000005
```

如果有更旧的检查点，会在这时被删除。

## 回放 WAL

我们首先从最后一个检查点按顺序对记录做迭代（有最大的数字编号的检查点就是最后的）。对于 `checkpoint.X`，`X` 告诉我们需要从哪一个 WAL 段继续回放，也就是 `X+1`。所以在上面的例子中，在回放完 `checkpoint.000003` 后，我们继续从 WAL 段 `000004` 继续回放。

你也许在想为什么我们需要在检查点中追踪段的数字编号，毕竟我们已经把在这之前的 WAL 段都删除了。这是因为，创建检查点和删除 WAL 的操作**不是原子性**的。在此期间任何事都可能发生而阻止了 WAL 段的删除。这种情况下我们不得不回放额外的本应该被删除的2/3的 WAL 段，这会使得回放变慢。

> 译注：试想下如果checkpoint目录没有带上编号作为后缀的话，而WAL段又没有成功删除，我们就无从知道哪些WAL段本应该删除，只好回放所有WAL段了。


对于单条记录，会执行如下操作：

1. `Series`：使用在记录中提到的相同的引用在 Head 中创建序列（这样后续我们能匹配上 samples）。对于相同的序列可能会有多个序列记录，Prometheus 会通过映射引用来处理这个问题。

2. `Samples`：把记录中的 samples 添加到 Head 中。记录中的引用表明了要添加到哪个序列中。如果对于该引用没有找到序列，那么会跳过这个 sample。

3. `Tombstones`：使用引用来确定序列，然后将墓碑恢复到 Head 中。

## 读写 WAL 的底层细节

当大规模的写入请求到来时，你想通过避免磁盘随机写来避免[写放大](https://en.wikipedia.org/wiki/Write_amplification)现象。另外，当读取记录时，你想确认它没有损坏（这在突然关机或者故障的磁盘上很可能发生）。

Prometheus 有一个 WAL 的通用实现，一条记录就是一个字节的切片，调用方需要注意对记录做编码。为解决上述的两个问题，WAL 包做了如下事情：

1. 数据在写入磁盘时一次写一页。一页长 32KiB。如果记录比 32KiB 大，那它会被分解为若干个小块，每一小块中有一个 WAL 记录的header，这样就可以知道这一小块属于记录的尾部，还是头部，还是中间（对于能够容纳在一页中的记录也有 WAL 记录的header）。

2. 记录的尾部追加了一个校验和，这就可以在读时检测数据是否损坏了。

WAL 包负责在回放遍历记录时，无缝地拼接记录的片段，并检查记录的校验和。

默认情况下，WAL 记录没有被严重压缩（或者根本不压缩）。所以 WAL 包提供了使用 [Snappy](https://en.wikipedia.org/wiki/Snappy_(compression)) 压缩记录的选项（现在默认开启）。这个信息存储在 WAL 记录头中，所以如果你计划启用或禁用压缩，压缩和未压缩的记录可以共存。

## 参考代码

将记录作为字节切片和做底层磁盘交互的 WAL 实现在 [`tsdb/wal/wal.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/wal/wal.go) 中。这个文件中有写字节记录和将记录作为字节切片来迭代的实现。

[`tsdb/record/record.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/record/record.go) 中包含了多种记录以及它们的编解码的逻辑。

检查点的逻辑在 [`tsdb/wal/checkpoint.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/wal/checkpoint.go) 中。

[`tsdb/head.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/head.go) 中包含了剩下的内容：

1. 创建并编码记录，调用 WAL 写入的逻辑

2. 调用创建检查点和 WAL 截断的逻辑

3. 回放 WAL 记录，将其解码并恢复到内存中状态。

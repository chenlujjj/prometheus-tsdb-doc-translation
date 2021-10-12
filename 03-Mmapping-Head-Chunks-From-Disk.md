# Prometheus TSDB (Part 3): Memory Mapping of Head Chunks from Disk

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-mmapping-head-chunks-from-disk/

## 简介

在第1篇文章中我提到一旦 chunk “满”时，它被刷到磁盘中并做内存映射。这有助于减少 Head block 的内存占用，并加速 WAL 的重放，这在第2篇文章中已经讨论过了。本文中我们将更加深入地探讨这在 Prometheus 中是如何设计的。

## 写 chunks

回忆下第1篇文章，当一个 chunk 满时，我们切割出一个新的 chunk，旧的 chunks 成为不可变的，只能对其读取（下图中黄色的块）。

![](https://ganeshvernekar.com/blog/img/tsdb3.svg)

我们并没有将旧的 chunk 存放在内存中，而是将它刷到磁盘，并在内存中存储它的引用以便后续访问它。

![](https://ganeshvernekar.com/blog/img/tsdb4.svg)

这个刷入的 chunk 也就是从磁盘内存映射的 chunk。**不可变性**是这里最重要的因素，否则对每个 sample 重写压缩的 chunks 太低效了。

## 磁盘上的格式

此格式也能在 [GitHub](https://github.com/prometheus/prometheus/blob/master/tsdb/docs/format/head_chunks.md) 上找到。

### 文件

这些 chunks (译注：指的是上文中提到的做内存映射的chunks)位于 `chunks_head` 目录中，其文件序列号和 WAL 类似（区别是从1开始）。例如：

```
data
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

文件最大大小是 128MiB。现在细看单个文件，它含有一个 8B 大小的 header。

```
┌──────────────────────────────┐
│  magic(0x0130BC91) <4 byte>  │
├──────────────────────────────┤
│    version(1) <1 byte>       │
├──────────────────────────────┤
│    padding(0) <3 byte>       │
├──────────────────────────────┤
│ ┌──────────────────────────┐ │
│ │         Chunk 1          │ │
│ ├──────────────────────────┤ │
│ │          ...             │ │
│ ├──────────────────────────┤ │
│ │         Chunk N          │ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
```

`Magic Number` 可以是任何能够用来唯一地确定该文件是内存映射的 head chunks 文件的数字。因为是我实现了这个特性，所以我把它设置为我的生日 :)。 `Chunk Format` 告诉我们如何解码文件中的 chunks。额外的填充(padding)是为了给未来可能需要的其他 header 选项留下位置。

文件 header 后，接下来就是 chunks 了。

### Chunks

单个 chunk 是这个样子：

```
┌─────────────────────┬───────────────────────┬───────────────────────┬───────────────────┬───────────────┬──────────────┬────────────────┐
| series ref <8 byte> | mint <8 byte, uint64> | maxt <8 byte, uint64> | encoding <1 byte> | len <uvarint> | data <bytes> │ CRC32 <4 byte> │
└─────────────────────┴───────────────────────┴───────────────────────┴───────────────────┴───────────────┴──────────────┴────────────────┘
```

`series ref` 就是我们在第2篇文章中讨论过的序列引用，它是用来访问内存中序列的序列 id。`mint` 和 `maxt` 是 chunk 中 samples 的最小和最大时间戳。`encoding` 是用来压缩 chunks 的编码格式。`len` 是接下来剩余的字节数。`data` 是压缩后的 chunk 的实际字节。

`CRC32` 是上述 chunk 的内容的校验和，用来检查数据的完整性。

## 读 chunks

对于每一个 chunk，Head block 存储了该 chunk 的 `mint` 和 `maxt` 以及用于访问它的内存中引用。

引用的长度是8字节。前4字节表示 chunk 所在的文件编号，后4字节表示 chunk 在该文件中的起始位置的偏移量（起始位置就是 `series ref` 的首字节）。如果 chunk 在文件 `00093` 中，而 `series ref` 从该文件的 `1234` 偏移量开始，那么这个 chunk 的引用就是 `(93 << 32) | 1234` （位左移然后做位或运算）。

我们将 `mint` 和 `maxt` 存储在 Head 中，这样我们在选择 chunk 时就不必访问磁盘了。当需要访问 chunk 时，我们只需根据引用访问编码格式和 chunk 数据。

在具体代码实现中，文件就像是一个字节切片（每个文件一个切片），通过在某个索引位置访问切片来获取 chunk 数据；在底层，OS 将内存中的切片映射到磁盘。[从磁盘内存映射](https://en.wikipedia.org/wiki/Memory-mapped_file)是一个 OS 特性，**它只会将磁盘的正在被访问的一部分加载到内存中，而不是整个文件**。

## 启动时重放

在第2篇文章中，我们讨论了WAL重放，我们重放每个 sample 来重新创建压缩的 chunk。由于磁盘上已经有了压缩的完整的 chunks，我们不需要重新创建这些 chunks 了；但是仍然需要从WAL中创建还没满的 chunks。现在，有了这些从磁盘内存映射的 chunks，重放过程如下所示。

在启动时，首先我们遍历 `chunks_head` 目录中的所有 chunks，并在内存中构建 `series ref -> [list of chunk references along with mint and maxt belonging to this series ref]` 的映射。

然后继续执行在第2篇文章中描述的WAL重放，但有一些修改:

* 当遇到 `Series` 记录时，在创建序列后，我们在上面的映射中查找序列引用，如果存在任何内存映射的 chunks，我们将这个chunks列表附加到这个序列。

* 当遇到 `Samples` 记录时，如果这个 sample 对应的序列有任何内存映射的 chunks，并且这个 sample 属于它所覆盖的时间范围，那么我们就跳过该 sample。如果没有，那么我们将该 sample 摄入到Head block 中。

## 带来的改进

这些额外的复杂度有什么用呢？我们明明可以将 chunks 存储在内存和WAL中。这个特性是最近在2020年加入的，我们来看看它带来了什么。（可以在这篇 [Grafana Labs 的博客](https://grafana.com/blog/2020/06/10/new-in-prometheus-v2.19.0-memory-mapping-of-full-chunks-of-the-head-block-reduces-memory-usage-by-as-much-as-40/)中看到一些基准测试的图）。

### 节省内存

如果必须将 chunk 存储在内存中，则它可能占用120到200字节(可能还会更多，视乎 samples 的压缩程度)。现在这被替换为24字节——每个 chunk 的引用、chunk 的最小时间和最大时间各占8字节。

> 译注：回忆下第一篇博客的内容，每个chunk中有120个 samples，粗略估算下就是120～200字节。

虽然这听起来像是80~90%的内存占用减少，但事实并非如此。Head 需要存储更多的东西，比如内存中的索引、所有的符号（标签值）等，以及TSDB的其他部分也会占用一些内存。

实际情况中，我们可以看到内存占用减少了15~50%，这取决于抓取 sample 的速率和创建新序列的速率(称为“churn”)。另一件需要注意的事情是，如果运行的一些查询命中了磁盘上的很多 chunks，那么它们需要被加载到内存中进行处理。所以这并不是绝对的内存占用峰值的减少。

### 更快的启动

WAL重放是启动过程中最慢的部分。主要是因为：(1) 从磁盘解码WAL记录和 (2) 从各个 samples 重新构建压缩的 chunks 是重放过程中速度较慢的部分。内存映射 chunks 的迭代相对较快。

我们需要检查所有的记录，所以不能避免对记录进行解码。如上文所诉，在回放中我们跳过了在内存映射 chunks 范围内的 samples。这样我们就避免了重新创建那些完整的压缩 chunks，因此节省了重放的一些时间。它可以减少15~30%的启动时间。

## 垃圾回收

内存中的垃圾回收发生在 Head 截断期间，它会丢弃比截断时间 `T` 更早的 chunk 的引用。但文件仍然存在于磁盘上。与WAL段一样，我们也需要定期删除旧的内存映射的文件。

对于每个存在的内存映射的 chunk 文件（也就意味着在 TSDB 中打开），我们将文件中存在的所有 chunks 的最大绝对时间存储在内存中。对于活动文件（也就是当前正在写入 chunks 的文件），当添加新的 chunks 时会更新内存中的最大时间。重启期间，当迭代所有内存映射的 chunks 时，会把文件的最大时间恢复到内存中。

所以当 Head 截断是发生在时间`T`之前的数据时，我们称之为以时间`T`对文件截断。最大时间小于`T`的文件（除了活动文件）会在这时被删除，同时会保留文件顺序（举个例子，如果有文件 `5, 6, 7, 8`，其中`5`和`7`在时间`T`之前，那么只有`5`会被删除，剩下的顺序将是`6, 7, 8`)。

在截断之后，我们关闭活动文件并开始一个新的文件，因为在小容量和小规模的设置中，可能需要很长时间才能达到文件的最大大小。因此，在这里滚动文件将有助于在下一次截断期间删除旧的 chunks。

## 代码参考

[`tsdb/chunks/head_chunks.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/chunks/head_chunks.go) 中有关于写 chunks 到磁盘，通过引用访问它，截断，处理文件，在 chunks 上做迭代的所有实现。

[`tsdb/head.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/head.go) 中把上述内容当作黑盒使用，把 chunks 从磁盘做内存映射。

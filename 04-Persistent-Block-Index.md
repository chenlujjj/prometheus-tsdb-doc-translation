# Prometheus TSDB (Part 4): Persistent Block and its Index

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-persistent-block-and-its-index/

## 简介

在前3篇文章中，我们已经讨论过了和（内存中的）Head block 有关的事情（在撰写本文时，又增加了更多有关 Head 的事情）。本文我们将深入探讨在磁盘上的持久化 block。

## 持久化 block 是什么 & 在何时创建的

磁盘上的一个 block 是由一定时间范围内的 chunks 和其索引组成的。它是一个包括了多个文件的目录。每个 block 有一个唯一的 ID，也就是 [ULID](https://github.com/oklog/ulid)。

block 有一个有趣的属性：其中的 samples 是**不可变**的。如果你想增加/删除/更新 samples，你必须带上必需的改动重新写整个 block，而新的 block 有新的 ID。这两个 blocks 之间没有任何联系。我们通过墓碑对 block 做删除操作，而不会涉及到 samples，这是因为在每次删除请求时重写 block 并不是一个合理的办法；对此我们将会在本文中做更多讨论。

在第1篇文章中我们看到，当 Head block 被时间跨度达到 `chunkRange*3/2` 的数据填充时，我们将第一个 `chunkRange` 的数据转化为一个持久化 block。

![](https://ganeshvernekar.com/blog/img/tsdb8.svg)

![](https://ganeshvernekar.com/blog/img/tsdb9.svg)

在 blocks 的上下文中，我们将 `chunkRange` 称为 `blockRange`。在 Prometheus 中，从 Head 上切下来的第一个 block 时间跨度默认是 `2h`。

TSDB 的整体图如下：

![](https://ganeshvernekar.com/blog/img/tsdb1.svg)

当 blocks 变老时，多个 blocks 会被压缩（或者合并）以形成一个新的更大的 block，而老的 blocks 会被删掉。所以共有2种创建 block 的方法：从 Head 和从已存在的 blocks。我们将在后续文章中讨论压缩。

## block 的内容

一个 block 由4部分组成：

1. `meta.json`（文件）：block的元数据。

2. `chunks`（目录）：包含原始的 chunks，没有 chunks 的任何元数据。

3. `index`（文件）：block 的索引。

4. `tombstones`（文件）：删除的标记，用来在查询 block 时排除一些 samples。

用 `01EM6Q6A1YPX4G9TEB20J22B2R` 这个 block ID 作为例子，下面是该 block 的文件：

```
data
├── 01EM6Q6A1YPX4G9TEB20J22B2R
|   ├── chunks
|   |   ├── 000001
|   |   └── 000002
|   ├── index
|   ├── meta.json
|   └── tombstones
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

接下来深入看看其中每一部分。

### 1. `meta.json`

此文件包含了 block 所有必需的元数据。这里有一个示例：

```
{
    "ulid": "01EM6Q6A1YPX4G9TEB20J22B2R",
    "minTime": 1602237600000,
    "maxTime": 1602244800000,
    "stats": {
        "numSamples": 553673232,
        "numSeries": 1346066,
        "numChunks": 4440437
    },
    "compaction": {
        "level": 1,
        "sources": [
            "01EM65SHSX4VARXBBHBF0M0FDS",
            "01EM6GAJSYWSQQRDY782EA5ZPN"
        ]
    },
    "version": 1
}
```

`version` 表示如何解析元数据文件。

尽管目录名称被设置为 ULID，但只有在 `meta.json` 文件中的 `ulid` 字段才会被当作有效的 ID，而目录名称可以是任意的。

`minTime` 和 `maxTime` 是 block 中所有 chunks 的最小和最大的绝对时间戳。

`stats` 表示了 block 中的序列数量，samples 数量和 chunks 数量。

`compaction` 表示了该 block 的历史。`level` 表示该 block 经过了多少代。 `sources` 表示该 block 是从哪些 blocks 创建而来的（也就是，哪些 blocks 合并得到了这个 block）。如果它是从 Head block 创建而来的，那么 `sources` 会被设置为它自己（也就是 `01EM6Q6A1YPX4G9TEB20J22B2R`）。

### 2. `chunks`

### 3. `index`

#### A. `TOC`

#### B. `Symbol Table`

#### C. `Series`

#### D. `Label Offset Table` 和 `Label Index i`

##### `Label Index i`


##### `Label Offset Table`

#### E. `Postings Offset Table` 和 `Postings i`

##### `Postings i`

##### `Postings Offset Table`


### 4. `tombstones`

## 尾声


## 代码参考

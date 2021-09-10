# Prometheus TSDB (Part 3): Memory Mapping of Head Chunks from Disk

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-mmapping-head-chunks-from-disk/

## 简介

在第1篇文章中我提到一旦 chunk “满”时，它被刷到磁盘中并做内存映射。这有助于减少 Head block 的内存占用，并加速 WAL 的重放，这在第2篇文章中已经讨论过了。 本文中我们将更加深入地探讨这在 Prometheus 中是如何设计的。

## 写 chunks

回忆下第1篇文章，当一个 chunk 满时，我们切割出一个新的 chunk，老的 chunks 成为不可变的，只能对其读取（下图中黄色的块）。

![](https://ganeshvernekar.com/blog/img/tsdb3.svg)

我们并没有将老的 chunk 放在内存中，而是将它刷到磁盘，并在内存中存储它的引用以便后续访问它。

![](https://ganeshvernekar.com/blog/img/tsdb4.svg)

这个刷入的 chunk 也就是从磁盘内存映射的 chunk。不可变性是这里最重要的因素，否则对每个 sample 重写压缩的 chunks 太低效了。

## 磁盘上的格式

此格式也能在 [GitHub](https://github.com/prometheus/prometheus/blob/master/tsdb/docs/format/head_chunks.md) 上找到。

### 文件


### Chunks


## 读 chunks


## 启动时重放



## 带来的改进

### 节省内存


### 更快的启动


## 垃圾回收

## 代码参考

`tsdb/chunks/head_chunks.go` has all the implementation of writing chunks to disk, accessing it using a reference, truncation, handling the files, and way to iterate over the chunks.

`tsdb/head.go` uses the above as a black box to memory-map its chunks from disk.
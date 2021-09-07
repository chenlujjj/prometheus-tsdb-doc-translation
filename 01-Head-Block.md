
# Prometheus TSDB (Part 1): The Head Block

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/

## 简介 

虽然 Prometheus 2.0是3年前发布的，但是网络上并没有多少资源来帮助理解它的TSDB。[Fabian 的博客](https://fabxc.org/tsdb/)比较high level，而代码仓库里的[格式文档](https://github.com/prometheus/prometheus/tree/master/tsdb/docs/format)更多是面向开发者的参考。

Prometheus的TSDB 最近吸引了许多新的贡献者，但是由于缺乏资料，如何理解它已经成为痛点之一。因此，我计划在一系列博客中详细讨论 TSDB 是如何工作的，并且会参考一些源码。

在这篇文章中，我主要讨论 TSDB 的**内存部分** —— **Head block**。在接下来的博客中，我将更深入地讨论其他组件，如 WAL 和它的checkpoint，chunks的memory-mapping是如何设计的，压缩，持久化block和它的索引，还有即将到来的对chunks的快照。


## 序言

Fabian 的博客可以帮助理解数据模型、核心概念以及 TSDB 整体是如何设计的。他还在2017年 PromCon 大会上就此发表了[演讲](https://www.youtube.com/watch?v=b_pEevMAC3I)。我建议你在阅读这篇文章之前先看看他的博客和演讲，这样可以建立一个良好的基础。

我在这篇文章中阐述的关于 Head 中的样本（sample）的生命周期的内容也可以在我的 [KubeCon 演讲](https://www.youtube.com/watch?v=suMhZfg9Cuk)中找到。

## 关于 TSDB 的简短概述

![](https://ganeshvernekar.com/blog/img/tsdb1.svg)

在上图中，Head block 是数据库的**内存部分**，灰色的block是位于磁盘上的持久化了的block，它们是**不可变**的。我们有 Write-Ahead-Log (WAL) 用于写入的持久性。一个到来的sample(图中用粉色框表示)首先会进入 Head block 并在内存中停留一段时间，然后会刷入到磁盘并进行**内存映射**(蓝色框)。当这些内存映射的chunks或内存中的chunks老化到某个程度时，它们会刷到磁盘，成为持久化的 block。进一步地，多个 blocks 会随着它们的老化而合并，并在它们超过保留期后最终被删除。

## Head 中 Sample 的一生

下面要讨论的内容是关于单个[时间序列](https://prometheus.io/docs/concepts/data_model/)的，也适用于所有的时间序列。

![](https://ganeshvernekar.com/blog/img/tsdb2.svg)

samples 被存储在名为“chunk”的压缩单元中。 当一个 sample 到来时，它被摄取到“活跃的chunk”（图中红色块）中。这也是我们**唯一**可以**主动写入数据**的单元。

在将 sample 提交到 chunk 中的同时，我们还将它记录在磁盘上的 Write-Ahead-Log (WAL)中(图中棕色块)，以保证持久性(这意味着即使机器突然崩溃，我们也可以从中恢复内存中的数据)。我将单独写一篇博文，讨论 Prometheus 是如何处理 WAL 的。

![](https://ganeshvernekar.com/blog/img/tsdb3.svg)

一旦 chunk 被装填到120个样本(或者)时间跨度达到 “chunk/block 范围”时(称之为`chunkRange`)，默认是2h，一个新的 chunk 被切割，旧的 chunk 被称为“满了”。本文中，我们假设抓取间隔（scrape interval）为15秒，因此120个sample(一个装满的chunk)的时间跨度为30分钟。

上图中有数字1的黄色块是刚刚被填满的 chunk，而红色块是新创建的 chunk。

## 更多需要注意/理解的事情

### 索引在哪里？

索引在内存中，以**反向索引（inverted index）**的形式存储。在 Fabian 的博文中可以看到更多关于索引的总体思路。当 Head block 压缩并创建持久化的 block 时，Head block 会被截断，删除老的 chunks，并且对索引会做垃圾回收（GC），删除不在 Head 中的时序条目。

### 处理重启

当 TSDB 需要重启（不管是优雅重启还是突然重启）时，它使用在磁盘上的内存映射的 chunks 和 WAL 来重放数据和事件，以此来重建在内存中的索引和 chunk。

## 代码参考

[`tsdb/db.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/db.go) 协调TSDB的整体运作。

对于本文中相关的部分，内存中 chunks 的摄取（ingestion）的核心逻辑都在 [`tsdb/head.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/head.go) 中，它将 WAL 和内存映射作为一个黑盒来使用。
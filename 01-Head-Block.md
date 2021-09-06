
# Prometheus TSDB (Part 1): The Head Block

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/

## 简介 

虽然 Prometheus 2.0是3年前发布的，但是网络上并没有多少资源来帮助理解它的TSDB。[Fabian 的博客](https://fabxc.org/tsdb/)比较high level，而代码仓库里的[格式文档](https://github.com/prometheus/prometheus/tree/master/tsdb/docs/format)更多是面向开发者的参考。

Prometheus的TSDB 最近吸引了许多新的贡献者，但是由于缺乏资料，如何理解它已经成为痛点之一。因此，我计划在一系列博客中详细讨论 TSDB 是如何工作的，并且会参考一些源码。

在这篇文章中，我主要讨论 TSDB 的**内存部分** —— **Head block**。在接下来的博客中，我将更深入地讨论其他组件，如 WAL 和它的checkpoint，chunks的memory-mapping是如何设计的，压缩，持久化block和它的索引，还有即将到来的对chunks的快照。


## 序言

Fabian 的博客可以帮助理解数据模型、核心概念以及 TSDB 整体是如何设计的。他还在2017年 PromCon 大会上就此发表了[演讲](https://www.youtube.com/watch?v=b_pEevMAC3I)。我建议你在阅读这篇文章之前先看看他的博客和演讲，这样可以建立一个良好的基础。
我在这篇文章中阐述的关于 Head 中的样本（sample）的生命周期的内容也可以在我的 [KubeCon 演讲](https://www.youtube.com/watch?v=suMhZfg9Cuk)中找到。
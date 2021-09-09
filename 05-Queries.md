# Prometheus TSDB (Part 5): Queries

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-queries

## 简介

在前四篇博文中，我们看到了数据是如何存储在 TSDB 中的。现在是时候了解如何查询数据了。本文中，我们将探讨对持久化 block 做的三种类型的查询，并简要介绍对 Head block 的查询。

第4篇文章是阅读本文的前提，它讲述了数据如何存储在持久化 block 中。

[2021-01-16 编辑]：更新了一些关于否定匹配器如何工作的细节。

## 开场白

不要将这里要讨论的查询和 PromQL 弄混淆。本文中我们将
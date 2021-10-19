# Prometheus TSDB (Part 5): Queries

原文地址：https://ganeshvernekar.com/blog/prometheus-tsdb-queries

## 简介

在前四篇博文中，我们看到了数据是如何存储在 TSDB 中的。现在是时候了解如何查询数据了。本文中，我们将探讨对持久化 block 做的三种类型的查询，并简要介绍对 Head block 的查询。

第4篇文章是阅读本文的前提，它讲述了数据如何存储在持久化 block 中。

[2021-01-16 编辑]：更新了一些关于否定匹配器如何工作的细节。

## 开场白

不要将这里要讨论的查询和 PromQL 弄混淆。本文中我们将看到用于从 TSDB 中获取原始数据的底层查询。PromQL 引擎执行这些 TSDB 查询来获得原始数据，然后在此基础上执行 PromQL 的逻辑。所以我们在讨论的内容比 PromQL 更底层。

## TSDB 查询的种类

在写作本文时，对于持久化 blocks 总共有3种类型的查询。

1. `LabelNames()`: 返回 block 中所有唯一的标签名。
   
2. `LabelValues(name)`: 返回在索引中标签名`name`对应的所有可能的标签值。

3. `Select([]matcher)`: 返回序列中符合给定的匹配器切片的 samples。稍后我们将详细讨论这些匹配器。

在对 block 执行任何查询之前，我们对 block 创建一个查询器 `Querier`，它有执行查询的最小时间(`mint`)和最大时间(`maxt`)。这个`mint`和`maxt`只适用于`Select` 查询，而其他两类查询总是查看 block 中的所有值。

在了解所有3种查询类型之后，我们将讨论如何组合来自多个 block 的结果。

## `LabelNames()`

该查询会返回 block 中存在的所有唯一的标签名。举例来说，在序列 `{a="b", c="d"}` 中，标签名是 `"a"`和 `"c"`。

在第4篇文章中提到，`Label Offset Table` 已经不再使用，写这个表只是为了向后兼容。因此，`LabelNames()` 和 `LabelValues()` 都使用 `Postings Offset Table`。

当 block 的索引在启动（或创建block）时被加载时，我们将标签名和**一些**标签值在 postings offset table 中位置的列表的映射 `map[labelName][]postingOffset` 存储起来（目前是每第32个，包括第一个和最后一个标签值）。只存储部分标签值有助于节省内存。这个映射是通过在加载block时遍历 `Postings Offset Table` 中的所有条目来创建的。

> 译注： 这一段话没看明白，需要结合代码梳理。

现在你能想到我们是如何获得标签名的 —— 只要迭代这个内存中的 map 的 key，就能获得标签名。它们在返回之前被排序。这对于 UI 上的查询自动补全建议非常有用。

## `LabelValues(name)`

如上所述，我们在内存中存储所有标签名的**第一个和最后一个标签值的位置**。因此，对于 `LabelValues(name)` 查询，我们先获取给定 `name` 的第一个和最后一个标签值位置，然后在这两个位置之间的磁盘上迭代，以获得该标签名的所有标签值。重新强调下：**一个标签名的所有标签值都按字典序存储在 `Postings Offset Table` 中**。

例如，如果 block 中的序列是 `{a="b1", c="d1"}`， `{a="b2", c="d2"}`和 `{a="b3", c="d3"}`，那么 `LabelValues("a")` 将生成 `["b1", "b2", "b3"]`， `LabelValues("c")` 将生成 `["d1", "d2", "d3"]`。

这再次有助于查询时的自动补全建议。

## `Select([]matcher)`

这个查询有助于从给定的匹配器所描述的序列中获得原始的 TSDB samples。在讨论这个查询之前，我们需要知道什么是匹配器。

### 匹配器

匹配器表示能够匹配上序列的标签名和值的组合。例如，匹配器 `a="b"` 表示选取所有包含标签对 `a="b"` 的序列。

总共有4种类型的匹配器。

1. 相等 `labelName="<value>"`: 标签名应该与给定值完全匹配。

2. 不等 `labelName!="<value>"`: 标签名称不应该与给定的值完全匹配。

3. 正则相等 `labelName=~"<Regex>"`: 标签名的标签值应该满足给定的正则表达式。

4. 正则不等 `labelName!~"<regex>"`: 标签名的标签值不应该满足给定的正则表达式。

`labelName` 是完整的标签名，不允许使用正则表达式。正则表达式匹配器应该匹配整个标签值，而不是部分，因为它是被当作 `^(?:<regex>)$` 形式来使用的。

假设序列是：

* s1 = `{job="app1", status="404"}`

* s2 = `{job="app2", status="501"}`

* s3 = `{job="bar1", status="402"}`

* s4 = `{job="bar2", status="501"}`

下面是一些匹配器的例子：

* `status="501"` -> (s2, s4)

* `status!="501"` -> (s1, s3)

* `job=~"app.*"` -> (s1, s2)

* `job!~"app.*"` -> (s3, s4)

当有多于1个匹配器时，它是所有匹配器之间的 AND 操作 (即交集)。

* `job=~"app.*", status="501"` -> (s1, s2) ∩ (s2, s4) -> (s2)

* `job=~"bar.*", status!~"5.."` -> (s3, s4) ∩ (s1, s3) -> (s3)

### 选择样本

第一步是获取匹配器匹配的序列。我们需要得到各个匹配器的匹配到的所有序列，然后对它们取交集。

在第4篇文章中，我们看到 "posting" 是序列ID，它告诉我们序列信息在索引中的位置。 `Postings Offset Table` 和 `Postings i` 一起给出了一个标签值对的所有 postings。

#### 为单个匹配器获取 postings

对于相等匹配器，假设是 `a="b"`，我们直接从 postings offset table 中获取它的 postings 列表位置。因为对于一个标签名只存储了它一部分标签值的位置，所以对标签名 `a` 来说，我们可以得到标签值 `"b"` 落在哪两个位置区间内，然后在这两个位置区间内迭代条目，直到找到 `"b"` 为止。offset table 中的 `a="b"` 条目指向一个 postings 列表，该列表是包含 `a="b"` 的所有序列的 id。如果在 offset table 中没有这样的条目，那么对于该匹配器就是一个空的 postings 列表。

> 译注：推测这里使用二分法搜索找到`"b"`。

对于正则相等匹配器 `a=~"<rgx>"`，我们必须遍历 `Postings Offset Table` 中 `a` 的所有标签值，并检查是否满足匹配条件。我们获取所有匹配条目的 postings 列表，并将其合并（求并集），以获得这个匹配器的有序 postings 列表。以上文中的 `job=~"app.*"` 为例，我们找到 `job="app1" -> (s1)` 和`job="app2" -> (s2)`，在合并后，我们得到 `job=~"app.*”-> (s1, s2)`。

对于不等匹配器 `a!="b"` 和正则不等匹配器 `a!~"<rgx>"`，情况有一点不同。我们先获得不等和正则不等对应的相等和正则相等匹配器（即 `a!="b"` 变成了 `a="b"`，`a!~"<rgx>"`变成了`a=~"<rgx>"`），因为实践中获取所有不匹配的东西可能会非常庞大。因此，不能在查询中使用单独的否定匹配器，**至少要有一个相等或正则相等匹配器**。转换后，我们对这些 postings 做集合减法。请看下面的例子。

> 译注：试了一下，比如查询 `{__name__!="up"}`，会报错 `Error executing query: invalid parameter "query": 1:1: parse error: vector selector must contain at least one non-empty matcher`，确实不行。

#### 多个匹配器的 postings

我们首先使用上面的步骤获得所有单个匹配器的 postings 列表。正如前面对匹配器的讨论那样，我们将它们做交集，最终得到满足所有匹配器的 postings 列表（序列）。注意，当我们有一个否定匹配器时，集合操作的变化。

`job=~"bar.*", status!~"5.*"`

-> `(job=~"bar.*") ∩ (status!~"5.*")`

-> `(job=~"bar.*") - (status=~"5.*")`

-> `((job="bar1") ∪ (job="bar2")) - (status="501")`

-> `((s3) ∪ (s4)) - (s2, s4)`

-> `(s3, s4) - (s2, s4)` -> `(s3)`


类似地，如果匹配器是 `a="b", c!=d, e=~"f.*", g!~"h.*"`，那么集合操作是 `((a="b") ∩ (e=~"f.*")) - (c="d") - (g=~"h.*")`。

#### 最终获得 samples

一旦我们有了匹配器对应的所有序列 id(postings)，我们只需要一个个地遍历它们并执行以下操作：

1. 找到 `Series` 表中由序列 id 表示的序列。

2. 从序列中选取与查询器指定的时间范围 `mint` 到 `maxt` 重叠的所有 chunk 的引用。

3. 创建一个迭代器，对 `chunks` 目录中的 chunks 做迭代，寻找在 `mint` 和 `maxt` 之间的 samples。

`Select([]matcher)` 最终返回与匹配器相匹配的所有序列的样本迭代器。这些序列根据标签对做了排序。

## 一些实现细节

* 获取一个匹配器的 postings 时，所有匹配的条目对应的 postings 不会**同时**载入到内存中。因为索引是从磁盘上内存映射的，postings 是被懒迭代的，然后合并得到最终的列表。

* `Select([]matcher)` 不会预先返回所有序列的所有 sample 迭代器；结果可能有成百上千个序列。它们遵循和上述相似的方式。返回一个迭代器，该迭代器逐个遍历序列，给出它的 sample 迭代器。当被请求时，这个 sample 迭代器会懒加载 chunks。

## 查询多个 blocks

当有多个 block 与查询器的`mint` 到 `maxt` 的区间重叠时，该查询器实际上是一个合并查询器，它对各个 block 分别查询。上述的3种查询实际上是这样工作的:

1. `LabelNames()`: 从所有 blocks 中获取已排序的标签名，然后做N路合并(N way merge)。

2. `LabelValues(name)`: 从所有 blocks 中获取标签值，然后做N路合并。

3. `Select([]matcher)`: 使用 Select 方法从所有 blocks 中获取序列迭代器，并以迭代方式进行“懒惰的”N路合并。这是可行的，因为每个序列迭代器按照标签对的顺序返回序列。

## 查询 Head block

Head block 在内存中存储了标签值对和所有的 postings 列表的整个映射（可以用 `map[labelName]map[labelValue]postingsList` 表示），因此在访问它们时没有什么特别的。执行这3类查询的剩余过程中，对映射和 postings 列表的处理方式和上述保持一致。

## 代码参考

[`tsdb/index/index.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/index/index.go) 中包含了对持久化 block 执行 `LabelNames()` 和 `LabelValues(name)` 查询和对给定的标签名和值获取合并的 postings 列表的代码。

[`tsdb/querier.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/querier.go) 中包含了对持久化 block 执行 `Select([]matcher)` 查询，包括从索引中获取 postings 列表前根据匹配器来过滤标签值的代码。 [`tsdb/chunks/chunks.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/chunks/chunks.go) 中包含了从磁盘上获取 chunks 的代码。

[`tsdb/head.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/head.go) 包含了对于 Head block 如何执行这3类查询的代码。

[`tsdb/db.go`](https://github.com/prometheus/prometheus/blob/master/tsdb/db.go) 和 [`storage/merge.go`](https://github.com/prometheus/prometheus/blob/master/storage/merge.go) 中，有在查询涉及到多个 blocks 时使用合并查询器的代码。

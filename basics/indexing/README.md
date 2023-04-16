---
description: Pinot 中的索引技术。
---

# Indexing

Pinot 支持以下索引技术：

* [Forward Index](forward-index.md)
  * Dictionary-encoded forward index with bit compression
  * Raw value forward index
  * Sorted forward index with run-length encoding
* [Inverted Index](inverted-index.md)
  * Bitmap inverted index
  * Sorted inverted index
* Star-tree Index
* [Bloom Filter](bloom-filter.md)
* [Range Index](range-index.md)
* Text Index
  * [Native Text Index](native-text-index.md)
  * [Text Search Support](text-search-support.md)
* Geospatial
* [JSON Index](json-index.md)
* [Timestamp Index](timestamp-index.md)

每一种索引技术在不同的场景中都有各自的优势。默认情况下，Pinot 为每一列创建一个**字典编码正排索引**。

## 启用索引

要为一个 Pinot 表创建索引有两种方式：

### 1. 在 Pinot Segment 生成期间，作为 ingestion 的一部分

通过在表配置中指定希望添加索引的列名来启用索引。可以在 [Table Config](https://docs.pinot.apache.org/configuration-reference/table) 部分以及上面的各小节中查看如何配置各种类型索引的更多细节。

### 2. 动态地添加或删除索引

索引也可以动态地在任何 pinot 分段中添加或删除，更新表配置到你需要的最新的索引集。

例如，如果你已经为 `foo` 列添加了倒排索引，并且现在想为 `bar` 列也添加相同的索引，你可以将表配置从下面这样：

```json
"tableIndexConfig": {
    "invertedIndexColumns": ["foo"],
    ...
}
```

更新成这样：

```json
"tableIndexConfig": {
    "invertedIndexColumns": ["foo", "bar"],
    ...
}
```

更新的索引配置需要在调用了 reload API 之后生效。这个 API 通过 Helix 将重新加载的信息发送到所有服务器，作为从本地分段添加或删除索引的一部分。这个改变不需要任何停机时间并且对查询完全透明。

当添加一个索引时，只有这个新的索引会被创建并且添加到现有的分段上；当删除一个索引时，该索引所有相关的状态会被从 Pinot 服务器一起清除。

你可以在 Swagger 页面的 Segments 部分中找到 reload API ：

```
curl -X POST \
  "http://localhost:9000/segments/myTable/reload" \
  -H "accept: application/json"
```

也可以在 Pinot UI 的[集群管理器](https://docs.pinot.apache.org/basics/components/exploring-pinot#cluster-manager)的特定表页面中找到该动作。

> 并非所有的索引都可以回顾性地应用于现有的分段。有关应用索引的更详细文档，请参阅 [Indexing FAQ](https://docs.pinot.apache.org/basics/getting-started/frequent-questions/ingestion-faq#indexing) 。

## 调整索引

对大部分用例来说，倒排索引提供了很好的性能，特别是你的用例没有严格的低延迟的需求时。

你应该首先尝试倒排索引，如果你觉得查询不够快速时，可以切换到高级索引，如排序索引或星树索引。

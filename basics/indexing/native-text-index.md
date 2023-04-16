---
description: 本页讨论了 Pinot 中的原生文本索引和相应的搜索功能。
---

# Native Text Index

## Pinot 中文本索引与搜索的历史

Pinot 通过为主要分段构建 Lucene 索引，作为 “sidecars” ，来支持文本索引和搜索。虽然这是一项很棒的技术，但它本质上限制了针对 Pinot 特定的文本搜索用例进行优化的途径。

## Pinot 搜索场景有什么不同？

Pinot，或任何其他 database/OLAP 引擎，是不需要遵守传统上由 FTS 引擎（如 ElasticSearch 和 Solr）使用的整个全文搜索 DSL 的。传统的 SQL 文本搜索用例中，大多数文本搜索由三种模式组成：前缀通配符 (prefix wildcard) 查询、后缀通配符 (postfix wildcard) 查询和术语 (term) 查询。

## Pinot 中的原生文本索引

原生文本索引是从头开始构建的，使用一个自定义的文本索引引擎，加上 Pinot 强大的倒排索引，以提供超快速的文本搜索体验。

## 原生文本索引的优势

对于上面提到的文本搜索用例，原生文本索引比基于 Lucene 的索引快 80-120% ，占用的磁盘空间也小了 40% 。

## 实时索引和搜索

原生文本索引支持的一个新特性是**实时文本搜索**。对于 REALTIME 表，本地文本索引允许在内存中对数据进行索引，同时支持在该索引上进行文本搜索。

从历史上看，大多数文本索引都依赖于先写入内存中的文本索引，然后封好 (sealed) ，然后才能进行搜索。这限制了搜索的新鲜度，最多只能接近实时。

原生文本索引带有一个自定义的内存文本索引，允许实时索引和搜索。

## 搜索原生文本索引

引入一个新函数 `TEXT_CONTAINS` ，用于支持在原生文本索引上进行文本搜索。

```SQL
SELECT COUNT(*) FROM Foo WHERE TEXT_CONTAINS (<column_name>, <search_expression>)
```

示例：

```SQL
SELECT COUNT(*) FROM Foo WHERE TEXT_CONTAINS (<column_name>, "foo.*")
SELECT COUNT(*) FROM Foo WHERE TEXT_CONTAINS (<column_name>, ".*bar")
SELECT COUNT(*) FROM Foo WHERE TEXT_CONTAINS (<column_name>, "foo")
```

`TEXT_CONTAINS` 可以使用标准布尔运算符进行组合：

```SQL
SELECT COUNT(*) FROM Foo WHERE TEXT_CONTAINS ("col1", "foo") AND TEXT_CONTAINS ("col2", "bar")
```

注意，`TEXT_CONTAINS` 目前支持正则表达式和术语查询，而且只能在原生索引上工作。

注意，`TEXT_CONTAINS` 支持标准正则表达式模式 (regex patterns) ，如 SQL 标准查询中 LIKE 所使用的那样，所以可能与从 Lucene 查询有一些句法变化。

## 创建原生文本索引

原生文本索引是 Pinot 支持的一种文本搜索索引，因此使用常规的表配置方式创建。为了表明索引类型是原生 (native) 的，需要在 fieldConfig 中指定一个额外的 property ：

```json
"fieldConfigList":[
  {
     "name":"text_col_1",
     "encodingType":"RAW",
     "indexTypes": ["TEXT"],
     "properties":{"fstType":"native"}
  }
]
```

---
description: 本页讨论了 Pinot 对文本搜索功能的支持。
---

# Text Search Support

## 为什么需要文本搜索？

Pinot 通过在 non-BLOB 列上的索引支持超快的查询处理。使用了精确匹配过滤器 (exact matching filters) 的查询可以通过字典编码、倒排索引和排序索引的组合高效地运行。

例如对下面的查询很有帮助：

```sql
SELECT COUNT(*) 
FROM Foo 
WHERE STRING_COL = 'ABCDCD' 
AND INT_COL > 2000
```

该查询分别对类型为 `STRING` 和 `INT` 的两列进行精确匹配。

然而对于属于 BLOB/CLOB 领域的任意文本数据，我们需要的不仅仅是精确匹配，用户感兴趣的是对 BLOB 类数据进行正则表达式 (regex) 、短语 (phrase) 和模糊 (fuzzy) 查询。在 0.3.0 版本之前，必须使用 [regexp\_like](https://apache-pinot.gitbook.io/apache-pinot-cookbook/pinot-user-guide/pinot-query-language#wild-card-match-in-where-clause-only) 来实现这一点，但它基于扫描，性能较差，并且没有实现如模糊搜索（编辑距离搜索）之类的功能。

在 0.3.0 版本中，我们增加了对文本索引的支持，以高效地对类型为大的文本 BLOB 的 `STRING` 列进行任意搜索。这可以通过使用内置函数 `TEXT_MATCH` 来实现。

```sql
SELECT COUNT(*) 
FROM Foo 
WHERE TEXT_MATCH (<column_name>, '<search_expression>')
```

其中 `<column_name>` 是创建了文本索引的列，而 `<search_expression>` 有以下类型：

| Search Expression Type | Example                                               |
| ---------------------- | ----------------------------------------------------- |
| Phrase query           | TEXT\_MATCH (\<column\_name>, '"distributed system"') |
| Term Query             | TEXT\_MATCH (\<column\_name>, 'Java')                 |
| Boolean Query          | TEXT\_MATCH (\<column\_name>, 'Java AND c++')         |
| Prefix Query           | TEXT\_MATCH (\<column\_name>, 'stream\*')             |
| Regex Query            | TEXT\_MATCH (\<column\_name>, '/Exception.\*/')       |

## 数据集示例

理想情况下，文本搜索应该用于：由于每个列值都是相当大的文本，导致执行标准过滤器操作 (EQUALITY, RANGE, BETWEEN) 不能满足需求的 `STRING` 列。

### Apache 访问日志

考虑以下来自Apache访问日志的片段，日志中的每一行都由任意数据（IP地址、url、时间戳、符号等）组成，且每一行都表示一个列值，这样的数据很适合进行文本搜索。

假设以下数据片段存储在 Pinot 表的 `ACCESS_LOG_COL` 列中：

```
109.169.248.247 - - [12/Dec/2015:18:25:11 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-
109.169.248.247 - - [12/Dec/2015:18:25:11 +0100] "POST /administrator/index.php HTTP/1.1" 200 4494 "http://almhuette-raith.at/administrator/" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
46.72.177.4 - - [12/Dec/2015:18:31:08 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
46.72.177.4 - - [12/Dec/2015:18:31:08 +0100] "POST /administrator/index.php HTTP/1.1" 200 4494 "http://almhuette-raith.at/administrator/" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
83.167.113.100 - - [12/Dec/2015:18:31:25 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
83.167.113.100 - - [12/Dec/2015:18:31:25 +0100] "POST /administrator/index.php HTTP/1.1" 200 4494 "http://almhuette-raith.at/administrator/" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
95.29.198.15 - - [12/Dec/2015:18:32:10 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
95.29.198.15 - - [12/Dec/2015:18:32:11 +0100] "POST /administrator/index.php HTTP/1.1" 200 4494 "http://almhuette-raith.at/administrator/" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
109.184.11.34 - - [12/Dec/2015:18:32:56 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
109.184.11.34 - - [12/Dec/2015:18:32:56 +0100] "POST /administrator/index.php HTTP/1.1" 200 4494 "http://almhuette-raith.at/administrator/" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
91.227.29.79 - - [12/Dec/2015:18:33:51 +0100] "GET /administrator/ HTTP/1.1" 200 4263 "-" "Mozilla/5.0 (Windows NT 6.0; rv:34.0) Gecko/20100101 Firefox/34.0" "-"
```

关于此数据的一些搜索查询示例：

**计算 GET 请求的数量**

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(ACCESS_LOG_COL, 'GET')
```

**计算 `/administrator/index.php` 的 POST 请求数量**

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(ACCESS_LOG_COL, 'post AND administrator AND index')
```

**计算使用 Firefox 浏览器发出的 `/administrator/index.php` 的 POST 请求数量**

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(ACCESS_LOG_COL, 'post AND administrator AND index AND firefox')
```

### 简历文本

考虑另一个简单的简历文本的例子，文件中的每一行都代表不同候选人简历中的技能数据。

假设以下数据片段存储在 Pinot 表的 `SKILLS_COL` 列中，且文本中的每一行表示一个列值。

```
Distributed systems, Java, C++, Go, distributed query engines for analytics and data warehouses, Machine learning, spark, Kubernetes, transaction processing
Java, Python, C++, Machine learning, building and deploying large scale production systems, concurrency, multi-threading, CPU processing
C++, Python, Tensor flow, database kernel, storage, indexing and transaction processing, building large scale systems, Machine learning
Amazon EC2, AWS, hadoop, big data, spark, building high performance scalable systems, building and deploying large scale production systems, concurrency, multi-threading, Java, C++, CPU processing
Distributed systems, database development, columnar query engine, database kernel, storage, indexing and transaction processing, building large scale systems
Distributed systems, Java, realtime streaming systems, Machine learning, spark, Kubernetes, distributed storage, concurrency, multi-threading
CUDA, GPU, Python, Machine learning, database kernel, storage, indexing and transaction processing, building large scale systems
Distributed systems, Java, database engine, cluster management, docker image building and distribution
Kubernetes, cluster management, operating systems, concurrency, multi-threading, apache airflow, Apache Spark,
Apache spark, Java, C++, query processing, transaction processing, distributed storage, concurrency, multi-threading, apache airflow
Big data stream processing, Apache Flink, Apache Beam, database kernel, distributed query engines for analytics and data warehouses
CUDA, GPU processing, Tensor flow, Pandas, Python, Jupyter notebook, spark, Machine learning, building high performance scalable systems
Distributed systems, Apache Kafka, publish-subscribe, building and deploying large scale production systems, concurrency, multi-threading, C++, CPU processing, Java
Realtime stream processing, publish subscribe, columnar processing for data warehouses, concurrency, Java, multi-threading, C++,
```

关于此数据的一些搜索查询示例：

**计算具有 "machine learning" 和 "gpu processing" 技能的候选人数量**

这是一个短语搜索，在文本中寻找短语 "machine learning" 和 "gpu processing" 的精确匹配，但不要求两者的顺序。

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Machine learning" AND "gpu processing"')
```

**计算具有 "distributed systems" 以及 'Java' 或 'C++' 技能的候选人数量**

精确匹配 "distributed systems" 和其他术语的组合搜索。

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"distributed systems" AND (Java C++)')
```

### 查询日志

考虑一个日志文件片段，其中包含数据库处理的 SQL 查询。文件中的每一行（每条查询）代表了 Pinot 表中 `QUERY_LOG_COL` 列的一个列值。

```
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1560988800000 AND 1568764800000 GROUP BY dimensionCol3 TOP 2500
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1560988800000 AND 1568764800000 GROUP BY dimensionCol3 TOP 2500
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1545436800000 AND 1553212800000 GROUP BY dimensionCol3 TOP 2500
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1537228800000 AND 1537660800000 GROUP BY dimensionCol3 TOP 2500
SELECT dimensionCol2, dimensionCol4, timestamp, dimensionCol5, dimensionCol6 FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1561366800000 AND 1561370399999 AND dimensionCol3 = 2019062409 LIMIT 10000
SELECT dimensionCol2, dimensionCol4, timestamp, dimensionCol5, dimensionCol6 FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1563807600000 AND 1563811199999 AND dimensionCol3 = 2019072215 LIMIT 10000
SELECT dimensionCol2, dimensionCol4, timestamp, dimensionCol5, dimensionCol6 FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1563811200000 AND 1563814799999 AND dimensionCol3 = 2019072216 LIMIT 10000
SELECT dimensionCol2, dimensionCol4, timestamp, dimensionCol5, dimensionCol6 FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1566327600000 AND 1566329400000 AND dimensionCol3 = 2019082019 LIMIT 10000
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1560834000000 AND 1560837599999 AND dimensionCol3 = 2019061805 LIMIT 0
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1560870000000 AND 1560871800000 AND dimensionCol3 = 2019061815 LIMIT 0
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1560871800001 AND 1560873599999 AND dimensionCol3 = 2019061815 LIMIT 0
SELECT count(dimensionCol2) FROM FOO WHERE dimensionCol1 = 18616904 AND timestamp BETWEEN 1560873600000 AND 1560877199999 AND dimensionCol3 = 2019061816 LIMIT 0
```

关于此数据的一些搜索查询示例：

**计算包含 "GROUP BY" 的查询数量**

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(QUERY_LOG_COL, '"group by"')
```

**计算包含 SELECT count... 模式的查询数量**

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(QUERY_LOG_COL, '"select count"')
```

**计算在 timestamp 列上使用了 BETWEEN 过滤并且使用了 GROUP BY 的查询数量**

```sql
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(QUERY_LOG_COL, '"timestamp between" AND "group by"')
```

后续[章节](text-search-support.md#bian-xie-wen-ben-sou-suo-cha-xun)会详细介绍关于每种查询的几个具体示例，以及使用 Pinot 编写文本搜索查询的分步指南。

## 当前限制

目前，我们支持的文本搜索有以下约束条件：

* 列类型应该是 `STRING`
* 列应该是单值的
* 目前不支持文本索引与其他 Pinot 索引共存

在即将发布的版本中，后面两个限制将很快被放松。

### 与其它索引共存

目前，Pinot 中的列可以是字典编码，也可以是 RAW 存储。此外，我们可以在字典编码的列上创建倒排索引，也可以在字典编码的列上创建排序索引。

文本索引是用户可以在 Pinot 表的每列上创建的索引类型的补充。但是，目前的实现仅支持 RAW 列上的文本索引。换句话说，要创建文本索引的列不应该是字典编码的。当我们在即将发布的版本中放松这一约束时，文本索引可以在字典编码的列上创建，该列还可以具有其他索引（倒排、排序等）。

## 如何启用文本索引？

和其他索引相同，用户可以通过表配置启用文本索引。作为文本搜索特性的一部分，我们还引入了一个新的通用方法来指定每一列的编码和索引信息。在[表配置](../../configuration-reference/table.md)页的 fieldConfigList 小节介绍了相关内容。

```javascript
"fieldConfigList":[
  {
     "name":"text_col_1",
     "encodingType":"RAW",
     "indexTypes":["TEXT"]
  },
  {
     "name":"text_col_2",
     "encodingType":"RAW",
     "indexTypes":["TEXT"]
  }
]
```

因为我们还没有移除老的指定索引信息的方式，为列添加文本索引还可以通过 `tableIndexConfig` 中的 `noDictionaryColumns` 指定：

```javascript
"tableIndexConfig": {
   "noDictionaryColumns": [
     "text_col_1",
     "text_col_2"
 ]}
```

The above mechanism can be used to configure text indexes in the following scenarios:
上面的机制可以用于在以下场景中配置文本索引：

* 新添加一个其中一列或多列启用了文本索引的表
* 在一个现有的表中添加一个启用了文本索引的新列
* 在一个现有的列上启用文本索引

{% hint style="info" %}
当使用文本索引时，我们建议将索引列添加到 `noDictionaryColumns` 下来减少不必要的存储开销。&#x20;

该配置属性的说明，请查看 [Raw value forward index](forward-index.md#原始值正排索引) 文档。
{% endhint %}


## 文本索引的创建过程

当通过表配置启用了一列或多列上的文本索引之后，我们的分段生成代码将读取配置并自动创建文本索引（每列）。这也是 Pinot 中其他索引的创建方式。文本索引既支持离线分段也支持实时分段。

### 文本解析和标记化

The original text document (a value in the column with text index enabled) is parsed, tokenized and individual "indexable" terms are extracted. These terms are inserted into the index.

Pinot's text index is built on top of Lucene. Lucene's **standard english text tokenizer** generally works well for most classes of text. We might want to build custom text parser and tokenizer to suit particular user requirements. Accordingly, we can make this configurable for the user to specify on per column text index basis.

There is a default set of "stop words" built in Pinot's text index. This is a set of high frequency words in English that are excluded for search efficiency and index size, including:

```
"a", "an", "and", "are", "as", "at", "be", "but", "by", "for", "if", "in", "into", "is", "it",
"no", "not", "of", "on", "or", "such", "that", "the", "their", "then", "than", "there", "these", 
"they", "this", "to", "was", "will", "with", "those"
```

Any occurrence of these words in will be ignored by the tokenizer during index creation and search.&#x20;

In some cases, users might want to customize the set. A good example would be when `IT` (Information Technology) appears in the text that collides with "it", or some context-specific words that are not informative in the search. To do this, one can config the words in `fieldConfig` to include/exclude from the default stop words:

```
"fieldConfigList":[
  {
     "name":"text_col_1",
     "encodingType":"RAW",
     "indexType":"TEXT",
     "properties": {
        "stopWordInclude": "incl1, incl2, incl3",
        "stopWordExclude": "it"
     }
  }
]
```

The words should be **comma separated** and in **lowercase**. Duplicated words in both list will end up get excluded.

## 编写文本搜索查询

A new built-in function TEXT\_MATCH has been introduced for using text search in SQL/PQL.

TEXT\_MATCH(text\_column\_name, search\_expression)

* text\_column\_name - name of the column to do text search on.
* search\_expression - search query

We can use TEXT\_MATCH function as part of our queries in the WHERE clause. Examples:

```sql
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(...)
SELECT * FROM Foo WHERE TEXT_MATCH(...)
```

We can also use the TEXT\_MATCH filter clause with other filter operators. For example:

```sql
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(...) AND some_other_column_1 > 20000
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(...) AND some_other_column_1 > 20000 AND some_other_column_2 < 100000
```

Combining multiple TEXT\_MATCH filter clauses

```sql
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(text_col_1, ....) AND TEXT_MATCH(text_col_2, ...)
```

TEXT\_MATCH can be used in WHERE clause of all kinds of queries supported by Pinot

* Selection query which projects one or more columns
  * User can also include the text column name in select list
* Aggregation query
* Aggregation GROUP BY query

The search expression (second argument to TEXT\_MATCH function) is the query string that Pinot will use to perform text search on the column's text index. \_\*\*\_Following expression types are supported

### **Phrase Query**

This query is used to do exact match of a given phrase. Exact match implies that terms in the user-specified phrase should appear in the exact same order in the original text document. Note that document is referred to as the column value.

Let's take the example of resume text data containing 14 documents to walk through queries. The data is stored in column named SKILLS\_COL and we have created a text index on this column.

```
Java, C++, worked on open source projects, coursera machine learning
Machine learning, Tensor flow, Java, Stanford university,
Distributed systems, Java, C++, Go, distributed query engines for analytics and data warehouses, Machine learning, spark, Kubernetes, transaction processing
Java, Python, C++, Machine learning, building and deploying large scale production systems, concurrency, multi-threading, CPU processing
C++, Python, Tensor flow, database kernel, storage, indexing and transaction processing, building large scale systems, Machine learning
Amazon EC2, AWS, hadoop, big data, spark, building high performance scalable systems, building and deploying large scale production systems, concurrency, multi-threading, Java, C++, CPU processing
Distributed systems, database development, columnar query engine, database kernel, storage, indexing and transaction processing, building large scale systems
Distributed systems, Java, realtime streaming systems, Machine learning, spark, Kubernetes, distributed storage, concurrency, multi-threading
CUDA, GPU, Python, Machine learning, database kernel, storage, indexing and transaction processing, building large scale systems
Distributed systems, Java, database engine, cluster management, docker image building and distribution
Kubernetes, cluster management, operating systems, concurrency, multi-threading, apache airflow, Apache Spark,
Apache spark, Java, C++, query processing, transaction processing, distributed storage, concurrency, multi-threading, apache airflow
Big data stream processing, Apache Flink, Apache Beam, database kernel, distributed query engines for analytics and data warehouses
CUDA, GPU processing, Tensor flow, Pandas, Python, Jupyter notebook, spark, Machine learning, building high performance scalable systems
Distributed systems, Apache Kafka, publish-subscribe, building and deploying large scale production systems, concurrency, multi-threading, C++, CPU processing, Java
Realtime stream processing, publish subscribe, columnar processing for data warehouses, concurrency, Java, multi-threading, C++,
C++, Java, Python, realtime streaming systems, Machine learning, spark, Kubernetes, transaction processing, distributed storage, concurrency, multi-threading, apache airflow
Databases, columnar query processing, Apache Arrow, distributed systems, Machine learning, cluster management, docker image building and distribution
Database engine, OLAP systems, OLTP transaction processing at large scale, concurrency, multi-threading, GO, building large scale systems
```

**Example 1 -** Search in SKILL\_COL column to look for documents where each matching document MUST contain phrase "distributed systems" as is

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Distributed systems"')
```

The search expression is '\\"Distributed systems\\"'

* The search expression is **always specified within single quotes** '\<your expression>'
* Since we are doing a phrase search, the **phrase should be specified within double quotes** inside the single quotes and the **double quotes should be escaped**
  * '\\"\<your phrase>\\"'

The above query will match the following documents:

```
Distributed systems, Java, C++, Go, distributed query engines for analytics and data warehouses, Machine learning, spark, Kubernetes, transaction processing
Distributed systems, database development, columnar query engine, database kernel, storage, indexing and transaction processing, building large scale systems
Distributed systems, Java, realtime streaming systems, Machine learning, spark, Kubernetes, distributed storage, concurrency, multi-threading
Distributed systems, Java, database engine, cluster management, docker image building and distribution
Distributed systems, Apache Kafka, publish-subscribe, building and deploying large scale production systems, concurrency, multi-threading, C++, CPU processing, Java
Databases, columnar query processing, Apache Arrow, distributed systems, Machine learning, cluster management, docker image building and distribution
```

But it won't match the following document:

```
Distributed data processing, systems design experience
```

This is because the phrase query looks for the phrase occurring in the original document **"as is"**. The terms as specified by the user in phrase should be in the **exact same order in the original document** for the document to be considered as a match.

**NOTE:** Matching is always done in a case-insensitive manner.

**Example 2 -** Search in SKILL\_COL column to look for documents where each matching document MUST contain phrase "query processing" as is

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"query processing"')
```

The above query will match the following documents:

```
Apache spark, Java, C++, query processing, transaction processing, distributed storage, concurrency, multi-threading, apache airflow
Databases, columnar query processing, Apache Arrow, distributed systems, Machine learning, cluster management, docker image building and distribution"
```

### **Term Query**

Term queries are used to search for individual terms

**Example 3 -** Search in SKILL\_COL column to look for documents where each matching document MUST contain the term 'java'

As mentioned earlier, the search expression is always within single quotes. However, since this is a term query, we don't have to use double quotes within single quotes.

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, 'Java')
```

### Composite Query using Boolean Operators

Boolean operators AND, OR are supported and we can use them to build a composite query. Boolean operators can be used to combine phrase and term queries in any arbitrary manner

**Example 4 -** Search in SKILL\_COL column to look for documents where each matching document MUST contain phrases "distributed systems" and "tensor flow". This combines two phrases using AND boolean operator

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Machine learning" AND "Tensor Flow"')
```

The above query will match the following documents:

```
Machine learning, Tensor flow, Java, Stanford university,
C++, Python, Tensor flow, database kernel, storage, indexing and transaction processing, building large scale systems, Machine learning
CUDA, GPU processing, Tensor flow, Pandas, Python, Jupyter notebook, spark, Machine learning, building high performance scalable systems
```

**Example 5 -** Search in SKILL\_COL column to look for documents where each document MUST contain phrase "machine learning" and term 'gpu' and term 'python'. This combines a phrase and two terms using boolean operator

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Machine learning" AND gpu AND python')
```

The above query will match the following documents:

```
CUDA, GPU, Python, Machine learning, database kernel, storage, indexing and transaction processing, building large scale systems
CUDA, GPU processing, Tensor flow, Pandas, Python, Jupyter notebook, spark, Machine learning, building high performance scalable systems
```

When using Boolean operators to combine term(s) and phrase(s) or both, please note that:

* The matching document can contain the terms and phrases in any order.
* The matching document may not have the terms adjacent to each other (if this is needed, please use appropriate phrase query for the concerned terms).

Use of OR operator is implicit. In other words, if phrase(s) and term(s) are not combined using AND operator in the search expression, OR operator is used by default:

**Example 6 -** Search in SKILL\_COL column to look for documents where each document MUST contain ANY one of:

* phrase "distributed systems" OR
* term 'java' OR
* term 'C++'.

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"distributed systems" Java C++')
```

We can also do grouping using parentheses:

**Example 7 -** Search in SKILL\_COL column to look for documents where each document MUST contain

* phrase "distributed systems" AND
* at least one of the terms Java or C++

In the below query, we group terms Java and C++ without any operator which implies the use of OR. The root operator AND is used to combine this with phrase "distributed systems"

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"distributed systems" AND (Java C++)')
```

### Prefix Query

Prefix searches can also be done in the context of a single term. We can't use prefix matches for phrases.

**Example 8 -** Search in SKILL\_COL column to look for documents where each document MUST contain text like stream, streaming, streams etc

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, 'stream*')
```

The above query will match the following documents:

```
Distributed systems, Java, realtime streaming systems, Machine learning, spark, Kubernetes, distributed storage, concurrency, multi-threading
Big data stream processing, Apache Flink, Apache Beam, database kernel, distributed query engines for analytics and data warehouses
Realtime stream processing, publish subscribe, columnar processing for data warehouses, concurrency, Java, multi-threading, C++,
C++, Java, Python, realtime streaming systems, Machine learning, spark, Kubernetes, transaction processing, distributed storage, concurrency, multi-threading, apache airflow
```

### Regular Expression Query

Phrase and term queries work on the fundamental logic of looking up the terms (aka tokens) in the text index. The original text document (a value in the column with text index enabled) is parsed, tokenized and individual "indexable" terms are extracted. These terms are inserted into the index.

Based on the nature of original text and how the text is segmented into tokens, it is possible that some terms don't get indexed individually. In such cases, it is better to use regular expression queries on the text index.

Consider server log as an example and we want to look for exceptions. A regex query is suitable for this scenario as it is unlikely that 'exception' is present as an individual indexed token.

Syntax of a regex query is slightly different from queries mentioned earlier. The regular expression is written between a pair of forward slashes (/).

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE text_match(SKILLS_COL, '/.*Exception/')
```

The above query will match any text document containing exception.

### Deciding Query Types

Generally, a combination of phrase and term queries using boolean operators and grouping should allow us to build a complex text search query expression.

The key thing to remember is that phrases should be used when the order of terms in the document is important and if separating the phrase into individual terms doesn't make sense from end user's perspective.

An example would be phrase "machine learning".

```sql
TEXT_MATCH(column, '"machine learning"')
```

However, if we are searching for documents matching Java and C++ terms, using phrase query "Java C++" will actually result in in partial results (could be empty too) since now we are relying the on the user specifying these skills in the exact same order (adjacent to each other) in the resume text.

```sql
TEXT_MATCH(column, '"Java C++"')
```

Term query using boolean AND operator is more appropriate for such cases

```sql
TEXT_MATCH(column, 'Java AND C++')
```
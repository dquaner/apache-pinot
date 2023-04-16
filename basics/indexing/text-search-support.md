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

```json
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

```json
"tableIndexConfig": {
   "noDictionaryColumns": [
     "text_col_1",
     "text_col_2"
 ]}
```

The above mechanism can be used to configure text indexes in the following scenarios: 上面的机制可以用于在以下场景中配置文本索引：

* 新添加一个其中一列或多列启用了文本索引的表
* 在一个现有的表中添加一个启用了文本索引的新列
* 在一个现有的列上启用文本索引

{% hint style="info" %}
当使用文本索引时，我们建议将索引列添加到 `noDictionaryColumns` 下来减少不必要的存储开销。

该配置属性的说明，请查看 [Raw value forward index](forward-index.md#yuan-shi-zhi-zheng-pai-suo-yin) 文档。
{% endhint %}

## 文本索引的创建过程

当通过表配置启用了一列或多列上的文本索引之后，我们的分段生成代码将读取配置并自动创建文本索引（每列）。这也是 Pinot 中其他索引的创建方式。文本索引既支持离线分段也支持实时分段。

### 文本解析和标记化

原始的文本记录（一个启用了文本索引的列中的值）会被解析，标记化 (tokenized) ，并且独立的“可索引的”短语会被提取并被添加到索引中。

Pinot 的文本索引建立在 Lucene 之上。Lucene 的**标准英语文本标记器**通常适用于大多数文本类。我们可能希望构建自定义文本解析器和标记器来满足特定的用户需求。因此，我们可以让用户在每个列文本索引的基础上指定这个可配置的参数。

Pinot 的文本索引内置了“停止词”的默认集合，这是一组基于搜索效率和索引大小而被排除在外的英语高频词，包括:

```
"a", "an", "and", "are", "as", "at", "be", "but", "by", "for", "if", "in", "into", "is", "it",
"no", "not", "of", "on", "or", "such", "that", "the", "their", "then", "than", "there", "these", 
"they", "this", "to", "was", "will", "with", "those"
```

在索引创建和搜索过程中，标记器将忽略这些词的出现。

在某些情况下，用户可能希望自定义“停止词”集合。一个很好的例子是，当 `IT`（信息技术）出现在与 "it" 冲突的文本中，或者一些上下文特定的单词在搜索中不包含实质信息。要做到这一点，可以配置 `fieldConfig` 来包含或排除默认停止词:

```json
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

单词应该用**逗号分隔**并**小写**，如果一个单词同时出现在在两个列表中，那么该单词将最终被排除。

## 编写文本搜索查询

引入新的内置函数 TEXT\_MATCH 以在 SQL/PQL 中只用文本搜索。

TEXT\_MATCH(text\_column\_name, search\_expression)

* text\_column\_name - 要进行文本搜索的列名
* search\_expression - 查询表达式

我们可以在 `WHERE` 子句中使用 TEXT\_MATCH 函数作为查询的一部分，例如：

```sql
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(...)
SELECT * FROM Foo WHERE TEXT_MATCH(...)
```

还可以与其他过滤操作一起使用 TEXT\_MATCH 过滤子句，例如：

```sql
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(...) AND some_other_column_1 > 20000
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(...) AND some_other_column_1 > 20000 AND some_other_column_2 < 100000
```

合并使用多个 TEXT\_MATCH 过滤子句：

```sql
SELECT COUNT(*) FROM Foo WHERE TEXT_MATCH(text_col_1, ....) AND TEXT_MATCH(text_col_2, ...)
```

TEXT\_MATCH 可以用在 Pinot 支持的各种查询的 `WHERE` 子句中：

* 投影一个或多个列的选择查询
  * 用户还可以在选择列表中包含文本列名
* 聚合查询
* 聚合分组查询

查询表达式（TEXT\_MATCH 函数的第二个参数）是 Pinot 用于在列的文本索引上执行文本搜索的查询字符串。支持以下表达式类型：

### 短语查询

用来精确匹配给定的短语。精确匹配意味着用户指定的短语中的单词应该在原始文本记录中以完全相同的顺序出现。注意，记录指列值。

让我们以包含 14 个记录的简历文本数据为例来过一遍短语查询。数据存储在名为 SKILLS\_COL 的列中，我们已经在该列上创建了一个文本索引。

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

**Example 1 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含短语 "distributed systems"

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Distributed systems"')
```

查询表达式为 `'"Distributed systems"'`

* search\_expression **总是在单引号中指定** '\<your expression>'
* 既然我们在做短语搜索，**短语必须在单引号中的双引号中指定**，并且**双引号应该进行转义**
  * '\\"\<your phrase>\\"'

上面的查询会匹配如下记录：

```
Distributed systems, Java, C++, Go, distributed query engines for analytics and data warehouses, Machine learning, spark, Kubernetes, transaction processing
Distributed systems, database development, columnar query engine, database kernel, storage, indexing and transaction processing, building large scale systems
Distributed systems, Java, realtime streaming systems, Machine learning, spark, Kubernetes, distributed storage, concurrency, multi-threading
Distributed systems, Java, database engine, cluster management, docker image building and distribution
Distributed systems, Apache Kafka, publish-subscribe, building and deploying large scale production systems, concurrency, multi-threading, C++, CPU processing, Java
Databases, columnar query processing, Apache Arrow, distributed systems, Machine learning, cluster management, docker image building and distribution
```

但不会匹配：

```
Distributed data processing, systems design experience
```

这是因为短语查询“**按原样**”查找出现在原始记录中的短语。用户在短语中指定的单词在原始记录中的顺序应完全相同，才能将该记录视为匹配。

**注意：** 匹配总是以不区分大小写的方式进行。

**Example 2 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含短语 "query processing"

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"query processing"')
```

上面的查询会匹配如下记录：

```
Apache spark, Java, C++, query processing, transaction processing, distributed storage, concurrency, multi-threading, apache airflow
Databases, columnar query processing, Apache Arrow, distributed systems, Machine learning, cluster management, docker image building and distribution"
```

### 单词查询

单词查询用来搜索独立的单词。

**Example 3 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含单词 'java'

如上所述，查询表达式总是被写在单引号中。因为这是一个单词查询，我们不需要在单引号中使用双引号。

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, 'Java')
```

### 使用布尔运算符的复合查询

支持布尔运算符 AND/OR ，并且我们可以使用它们构建一个复杂查询。布尔运算符可以用来以任意方式组合短语和单词查询。

**Example 4 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含短语 "distributed systems" 和 "tensor flow"。使用 AND 布尔操作符组合两个短语：

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Machine learning" AND "Tensor Flow"')
```

上面的查询将匹配如下记录：

```
Machine learning, Tensor flow, Java, Stanford university,
C++, Python, Tensor flow, database kernel, storage, indexing and transaction processing, building large scale systems, Machine learning
CUDA, GPU processing, Tensor flow, Pandas, Python, Jupyter notebook, spark, Machine learning, building high performance scalable systems
```

**Example 5 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含短语 "machine learning" 和单词 'gpu'，'python'。使用布尔操作符 AND 组合一个短语和单词：

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Machine learning" AND gpu AND python')
```

上面的查询将匹配如下记录：

```
CUDA, GPU, Python, Machine learning, database kernel, storage, indexing and transaction processing, building large scale systems
CUDA, GPU processing, Tensor flow, Pandas, Python, Jupyter notebook, spark, Machine learning, building high performance scalable systems
```

当使用布尔运算符组合术语和短语或两者时，请注意：

* 匹配的记录可以以任何顺序包含单词和短语。
* 相匹配的记录中单词可能不相邻（如有需要，请使用适当的短语查询有关单词）。

操作符 OR 的使用是隐式的。换句话说，如果搜索表达式中的短语或术语没有使用 AND 操作符组合起来，是默认使用 OR 操作符连接的。

**Example 6 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含以下任一：

* 短语 "distributed systems" 或
* 单词 'java' 或
* 单词 'C++'

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"distributed systems" Java C++')
```

还可以使用括号进行分组：

**Example 7 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含

* 短语 "distributed systems" 以及
* Java 或 C++ 中至少一个

在下面的查询中，我们在没有使用任何操作符的情况下将单词 Java 和 C++ 进行了分组，而事实上隐式使用了 OR ；外层的操作符 AND 用来组合该分组和短语 "distributed systems" 。

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"distributed systems" AND (Java C++)')
```

### 前缀查询

前缀搜索也可以在单个术语的上下文中进行，但我们不能对短语使用前缀匹配。

**Example 8 -** 在 SKILL\_COL 列中搜索，每个匹配记录必须包含如 stream，streaming，streams 等的文本：

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, 'stream*')
```

上面的查询将匹配如下记录：

```
Distributed systems, Java, realtime streaming systems, Machine learning, spark, Kubernetes, distributed storage, concurrency, multi-threading
Big data stream processing, Apache Flink, Apache Beam, database kernel, distributed query engines for analytics and data warehouses
Realtime stream processing, publish subscribe, columnar processing for data warehouses, concurrency, Java, multi-threading, C++,
C++, Java, Python, realtime streaming systems, Machine learning, spark, Kubernetes, transaction processing, distributed storage, concurrency, multi-threading, apache airflow
```

### 正则表达式查询

短语和单词查询的基本逻辑是在文本索引中查找单词（又名 tokens）。原始文本记录（启用了文本索引的列中的值）将被解析、标记化、提取单个“可索引”项并插入到索引中。

根据原始文本的性质以及文本划分为标记的规则，有些单词可能不会单独被索引。在这种情况下，最好对文本索引使用正则表达式查询。

以服务器日志为例，我们希望查找异常。正则表达式查询适用于这种情况，因为 'exception' 不太可能作为单独的索引令牌出现。

正则表达式查询的语法与前面提到的查询略有不同。正则表达式被写入一对正斜杠 (/) 之间。

```sql
SELECT SKILLS_COL 
FROM MyTable 
WHERE text_match(SKILLS_COL, '/.*Exception/')
```

上面的查询将匹配任何包含 exception 的文本记录。

### 选择查询类型

通常，使用布尔运算符和分组组合短语和单词查询允许我们构建复杂的文本搜索查询表达式。

关键是记住：如果记录中单词顺序很重要，且从最终用户的角度看将短语分离为单独的单词没有意义时，应该使用短语查询。

短语 "machine learning" 的例子：

```sql
TEXT_MATCH(column, '"machine learning"')
```

然而，如果我们正在搜索匹配 Java 和 C++ 术语的记录，使用短语查询 "Java C++" 实际上会导致只得到部分结果（也可能结果为空），因为现在我们依赖于用户在简历文本中以完全相同的顺序（并且彼此相邻）指定这些技能。

```sql
TEXT_MATCH(column, '"Java C++"')
```

使用 AND 操作符的单词查询更适合这种场景：

```sql
TEXT_MATCH(column, 'Java AND C++')
```

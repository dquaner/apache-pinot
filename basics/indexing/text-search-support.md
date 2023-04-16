# Text Search Support

## 为什么需要文本搜索？

Pinot 通过在 non-BLOB 列上的索引支持超快的查询处理。使用了精确匹配过滤器 (exact matching filters) 的查询可以通过字典编码、倒排索引和排序索引的组合高效地运行。

例如对下面的查询很有帮助：

```SQL
SELECT COUNT(*) 
FROM Foo 
WHERE STRING_COL = 'ABCDCD' 
AND INT_COL > 2000
```

该查询分别对类型为 `STRING` 和 `INT` 的两列进行精确匹配。

然而对于属于 BLOB/CLOB 领域的任意文本数据，我们需要的不仅仅是精确匹配，用户感兴趣的是对 BLOB 类数据进行正则表达式 (regex) 、短语 (phrase) 和模糊 (fuzzy) 查询。在 0.3.0 版本之前，必须使用 [regexp\_like](https://apache-pinot.gitbook.io/apache-pinot-cookbook/pinot-user-guide/pinot-query-language#wild-card-match-in-where-clause-only) 来实现这一点，但它基于扫描，性能较差，并且没有实现如模糊搜索（编辑距离搜索）之类的功能。

在 0.3.0 版本中，我们增加了对文本索引的支持，以高效地对类型为大的文本 BLOB 的 `STRING` 列进行任意搜索。这可以通过使用内置函数 `TEXT_MATCH` 来实现。

```SQL
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

## Apache 访问日志

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

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(ACCESS_LOG_COL, 'GET')
```

**计算 `/administrator/index.php` 的 POST 请求数量**

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(ACCESS_LOG_COL, 'post AND administrator AND index')
```

**计算使用 Firefox 浏览器发出的 `/administrator/index.php` 的 POST 请求数量**

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(ACCESS_LOG_COL, 'post AND administrator AND index AND firefox')
```

## 简历文本

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

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"Machine learning" AND "gpu processing"')
```

**计算具有 "distributed systems" 以及 'Java' 或 'C++' 技能的候选人数量**

精确匹配 "distributed systems" 和其他术语的组合搜索。

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(SKILLS_COL, '"distributed systems" AND (Java C++)')
```

## 查询日志

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

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(QUERY_LOG_COL, '"group by"')
```

**计算包含 SELECT count... 模式的查询数量**

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(QUERY_LOG_COL, '"select count"')
```

**计算在 timestamp 列上使用了 BETWEEN 过滤并且使用了 GROUP BY 的查询数量**

```SQL
SELECT COUNT(*) 
FROM MyTable 
WHERE TEXT_MATCH(QUERY_LOG_COL, '"timestamp between" AND "group by"')
```

后续[章节](./#编写文本搜索查询)会详细介绍关于每种查询的几个具体示例，以及使用 Pinot 编写文本搜索查询的分步指南。

## 当前限制

目前，我们支持的文本搜索有以下约束条件：

* 列类型应该是 `STRING`
* 列应该是单值的
* 目前不支持文本索引与其他 Pinot 索引共存

在即将发布的版本中，后面两个限制将很快被放松。

## 与其它索引共存

目前，Pinot 中的列可以是字典编码，也可以是 RAW 存储。此外，我们可以在字典编码的列上创建倒排索引，也可以在字典编码的列上创建排序索引。

文本索引是用户可以在 Pinot 表的每列上创建的索引类型的补充。但是，目前的实现仅支持 RAW 列上的文本索引。换句话说，要创建文本索引的列不应该是字典编码的。当我们在即将发布的版本中放松这一约束时，文本索引可以在字典编码的列上创建，该列还可以具有其他索引（倒排、排序等）。

## 如何启用文本索引？

## 文本索引的创建

## 编写文本搜索查询

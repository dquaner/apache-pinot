# Forward Index

每一列的值都存储在一个正排索引中，正排索引有三种类型:

*   [Dictionary encoded forward index](forward-index.md#wei-ya-suo-zi-dian-bian-ma-zheng-pai-suo-yin-mo-ren)

    构建一个将唯一 id 映射到列中的每个唯一值的字典，以及包含位压缩 id 的正排索引。
*   [Sorted forward index](forward-index.md#yun-hang-chang-du-bian-ma-pai-xu-zheng-pai-suo-yin)

    构建一个将每个唯一值映射到一个 start-end document id 对的字典，以及在字典编码上的正排索引。
*   [Raw value forward index](forward-index.md#yuan-shi-zhi-zheng-pai-suo-yin)

    直接根据列值构建正排索引。

为了节省分段存储空间，现在可以在创建表时[禁用](forward-index.md#jin-yong-zheng-pai-suo-yin)正排索引。

## 位压缩字典编码正排索引（默认）

一列中的每一个唯一值都会被分配一个 id ，并且创建一个字典来映射 id 到唯一值。正排索引会存储位压缩后的 ids 而不是值本身。如果这一列只有很少的唯一值，那么字典编码可以显著提高空间效率。

下面的图片展示了为 `integer` 和 `string` 类型的两列创建的字典编码。对 `colA` 来说，字典编码为重复值节省了大量空间。

而对 `colB` 来说，可以看到该列没有重复的数据，在列中有很多唯一值的情况下，字典编码的压缩作用就不明显了；并且对于 `string` 类型，Pinot 选择最长字符串的长度作为字典的定长数组的值的长度，因此如果一个字符串列有大量的唯一值，填充开销就会很高。

![Dictionary encoded forward index](../images/Indexing\_dictionary-encoded-forward-index-with-bit-compression.webp)

## 运行长度编码排序正排索引

当一个列被物理排序时，Pinot 在字典编码的基础上使用运行时长度编码排序正排索引。Pinot 将为每个值存储一对开始和结束文档 id ，而不是为每个文档 id 保存字典 id 。

![Sorted forward index](../images/Indexing\_sorted-forward-index.png)

为简单起见，上图不包括字典编码层。

排序正排索引同时具有良好的压缩和数据局部性的优点。排序正排索引也可以用作倒排索引。

### Real-time tables

在表配置中设置排序索引：

```json
{
    "tableIndexConfig": {
        "sortedColumn": [
            "column_name"
        ],
        ...
    }
}
```

> **注意：** 一个 Pinot 表只能有一个排序的列。

实时数据接收将在生成分段 (segment) 时根据 `sortedColumn` 对数据进行排序 - 你不需要预先对数据进行排序。

当提交一个分段时，Pinot 将传递每个列中的数据，并为包含排序数据的所有列创建一个排序索引，即使它们没有指定为 `sortedColumn` 。

### Offline tables

对于离线数据接收，Pinot 将传递每个列中的数据，并为包含已排序数据的列创建排序索引。这意味着，如果你希望某一列具有排序索引，则需要在将数据摄取到 Pinot 之前按该列对数据进行排序。

如果你正在接收多个分段 (segments) 的数据，需要确保数据在每个段内已排好序 - 你不需要跨分段对数据进行排序。

### 检查排序状态

你可以运行以下命令查看分段中某列的排序状态：

```sh
$ grep memberId <segment_name>/v3/metadata.properties | grep isSorted
column.memberId.isSorted = true
```

另外，对于离线表以及实时表中的已提交段，你可以从 `getServerMetadata` 端点检索排序状态。以 [Batch Quick Start](https://docs.pinot.apache.org/basics/getting-started/quick-start#batch) 为例:

```bash
curl -X GET \
  "http://localhost:9000/segments/baseballStats/metadata?columns=playerID&columns=teamID" \
  -H "accept: application/json" 2>/dev/null | \
  jq -c '.[] | . as $parent |  .columns[] | [$parent .segmentName, .columnName, .sorted]'
```

```
["baseballStats_OFFLINE_0","teamID",false]
["baseballStats_OFFLINE_0","playerID",false]
```

## 原始值正排索引

原始值正排索引直接存储值而不是 ids 。

如果没有字典，每次取值时都可以跳过字典查找步骤；此外，索引还可以利用值的良好局部性，从而提高扫描大量值的性能。

原始值正排索引适用于具有大量唯一值的列，这种情况下字典编码没能提供太多压缩效率。

如下图所示，使用字典编码将需要大量随机访问内存来进行字典查找；而使用原始值正排索引，可以按顺序扫描值，如果应用得当，可以提高查询性能。

![raw value forward index](../images/Indexing\_raw-value-forward-index.webp)

可以在 [table config](https://docs.pinot.apache.org/configuration-reference/table) 中配置原始值正排索引：

```json
{
    "tableIndexConfig": {
        "noDictionaryColumns": [
            "column_name",
            ...
        ],
        ...
    }
}
```

## 字典编码 vs 原始值

当确定一个列应该使用字典编码还是原始值编码时，可以参考下面的比较表格：

| 字典编码                 | 原始值                            |
| -------------------- | ------------------------------ |
| 在低到中等基数时提供压缩         | 消除填充开销                         |
| 允许索引（特别是 inv 索引）     | 没有 inv 索引（只有 JSON/Text/FST 索引） |
| 增加了一级解引用，所以增加了磁盘寻径开销 | 消除了额外的解引用，所以当要搜索的文档连续时性能很好     |
| 对于字符串，填充以使字典中的所有值都等长 | 所选文档的块解压开销没有空间局部性              |

## 禁用正排索引

一般来说，正排索引是所有磁盘分段 (segment) 文件格式中所有列上的强制索引。但是，**某些列只作为所有查询的** `WHERE` **子句中的筛选使用**，在这种情况下，正排索引是不必要的，因为分段中的其他索引和结构可以提供所需的 SQL 查询功能。正排索引在这种情况下只会占用额外的存储空间，理想情况下可以将其释放。

因此，为了给用户提供一个节省存储空间的选项，Pinot 现在可以选择禁用正排索引，但是有以下限制：

* 仅支持不可变（offline）分段。
* 如果该列上有范围索引，那么要求该列必须为 single-value 类型并且范围索引使用版本2。
* 在一行中有副本的 MV (multi-value) 列会在重新生成正排索引时丢失掉重复的条目；MV 行的数据顺序也会在重新生成索引时发生变化。因此在这种场景下，需要回填（以保存副本或数据顺序）。
* 如果需要在重载时支持再次生成正排索引（即为禁用了正排索引的列重新启用正排索引），则必须在该列上启用字典和倒排索引。

排序的列 (Sorted columns) 允许禁用正排索引，但此操作实际上无效并且仍会创建索引（它将同时用作正排索引和倒排索引 ）。

要禁用给定列的正排索引，需要修改表配置中的 `fieldConfigList` ，如下所示:

```json
"fieldConfigList":[
  {
     "name":"columnA",
     "encodingType":"DICTIONARY",
     "indexTypes":["INVERTED"],
     "properties": {
        "forwardIndexDisabled": "true"
      }
  }
]
```

必须执行表重载 (reload) 操作才能使上述配置生效。启用/禁用列上的其他索引可以通过常用的[表配置](https://docs.pinot.apache.org/configuration-reference/table)选项完成。

相反，可以通过从 `fieldConfigList.properties` 中移除 `forwardIndexDisabled` 属性并重新加载分段来重新生成被禁用的正排索引。只有在为列启用了字典和倒排索引时，才能重新生成正排索引。如果其中一个被禁用，那么生成正排索引的唯一方法是通过离线作业 (offline jobs) 重新生成分段并重新推送/刷新数据。

> **警告:**\
> 对于多值 (MV) 列，在为禁用了正排索引的列重新生成正排索引后，以下不变量 (invariants) 不能被维护：
>
> * 一行中 MV 值的顺序保证
> * 如果一个 MV 行中的项有副本，副本将丢失；请通过离线作业重新生成分段，并重新推送/刷新数据，以获得带有副本的原始 MV 数据。
>
> 我们将在未来努力去除第二个不变量。

### 不支持的查询

假如禁用了列 `columnA` 的正排索引将导致如下的查询失败：

#### **Select**

即使在 `columnA` 上添加了筛选器 (filters) ，禁用了正排索引的列也不能出现在 `SELECT` 子句中。

失败示例：

```sql
SELECT columnA
FROM myTable
WHERE columnA = 10
```

```sql
SELECT *
FROM myTable
```

#### **Group By & Order By**

在 `GROUP BY` 和 `ORDER BY` 子句中不能出现禁用了正排索引的列。它们也不能成为 `HAVING` 从句的一部分。

失败示例：

```sql
SELECT SUM(columnB)
FROM myTable
GROUP BY columnA
```

```sql
SELECT SUM(columnB), columnA
FROM myTable
GROUP BY columnA
ORDER BY columnA
```

```sql
SELECT MIN(columnA)
FROM myTable
GROUP BY columnB
HAVING MIN(columnA) > 100
ORDER BY columnB
```

#### **Aggregation Queries**

当正排索引被禁用时，聚合函数的一个子集可以工作，例如 `MIN` ，`MAX` ，`DISTINCTCOUNT` ，`DISTINCTCOUNTHLL` 等等；其他聚合函数将不能工作，如下面所示。

失败示例：

```sql
SELECT SUM(columnA), AVG(columnA)
FROM myTable
```

```sql
SELECT MAX(ADD(columnA, columnB))
FROM myTable
```

#### **Distinct**

禁用正排索引的列不能出现在 `SELECT DISTINCT` 子句中。

失败示例：

```sql
SELECT DISTINCT columnA
FROM myTable
```

#### **Range Queries**

要在筛选 (filter) 子句包含 `>` 、`<` 、`>=` 、`<=` 等操作符的单值列上运行查询，必须提供版本2的范围索引。如果没有范围索引，下面的查询将失败：

```sql
SELECT columnB
FROM myTable
WHERE columnA > 1000
```

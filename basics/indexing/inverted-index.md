# Inverted Index

## Bitmap inverted index

当为列启用倒排索引时，Pinot 维护从每个值到行位图 (bitmap) 的映射，这使得值查找仅需要花费常量 (constant) 时间。如果你有一个经常用于过滤 (filtering) 的列，那么添加倒排索引将大大提高性能。

倒排索引可以通过在表配置做如下设置来添加:

```javascript
{
    "tableIndexConfig": {
        "invertedIndexColumns": [
            "column_name",
            ...
        ],
        ...
    }
}
```

## Sorted inverted index

倒排索引可以直接使用排序正排索引 (sorted forward index) ，查找时间为 `log(n)` ，并且可以从数据局部性中获益。

对于下面的示例，如果查询在 `memberId` 上有过滤，Pinot 将对 `memberId` 值执行二分 (binary) 搜索，以找到对应过滤值的 `docid` 对。如果查询在过滤后需要扫描其他列的值，那么 `docId` 对范围内的值将被定位在一起，这意味着我们可以从数据局部性中获益。

![Sorted inverted index](../images/Indexing\_sorted-inverted-index.png)

排序索引的性能要比倒排索引好得多，但它只能应用于每个表的一个列上。当使用倒排索引的查询性能不够好，并且大多数查询都是在同一列上进行过滤 (e.g. memberId) 时，排序索引可以提高查询性能。

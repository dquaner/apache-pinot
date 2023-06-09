# Range Index

范围索引帮助你在涉及范围筛选的查询中获得更好的性能。它将对如下的查询有帮助：

```sql
SELECT COUNT(*) 
FROM baseballStats 
WHERE hits > 11
```

范围索引是倒排索引的一种变体，在倒排索引中，我们创建的不是从值到列的映射，而是从值的范围到列的映射。可以在表配置中添加如下配置来使用范围索引：

```javascript
{
    "tableIndexConfig": {
        "rangeIndexColumns": [
            "column_name",
            ...
        ],
        ...
    }
}
```

字典编码和原值编码列都支持范围索引。

## 什么时候应该使用范围索引？

一个好的经验法则是，当你想对具有大量唯一值的度量 (metric) 列应用范围谓词 (predicates) 时，为此列添加范围索引。如果为这些列添加倒排索引将创建一个非常大的索引，这在存储和性能方面都是低效的。

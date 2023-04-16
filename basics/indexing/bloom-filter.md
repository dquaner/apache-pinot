# Bloom Filter

Bloom Filter 帮助剔除那些不包含任何满足 `EQUALITY` 谓词的条目的分段。

它在如下查询中有用：

```sql
SELECT COUNT(*) 
FROM baseballStats 
WHERE playerID = 12345
```

布隆过滤器有 3 个配置参数：

* `fpp` : False Positive Probability (from 0 to 1，默认为 0.05) ；fpp 越低，过滤器的准确度越高，但是同时会增加占用的空间。
* `maxSizeInBytes` : Maximum size (默认不受限制) ；如果某个 fpp 生成的过滤器大小大于该最大值，Pinot 将调大 fpp 以使过滤器大小保持在此限制内。
* `loadOnHeap` : 是否使用堆内存 (heap) 或堆外 (off-heap) 内存加载布隆过滤器 (默认为 false) 。

有两种方法在表配置中配置布隆过滤器:

1. 默认设置

```javascript
{
  "tableIndexConfig": {
    "bloomFilterColumns": [
      "playerID",
      ...
    ],
    ...
  },
  ...
}
```

2. 自定义参数

```javascript
{
  "tableIndexConfig": {
    "bloomFilterConfigs": {
      "playerID": {
        "fpp": 0.01,
        "maxSizeInBytes": 1000000,
        "loadOnHeap": true
      },
      ...
    },
    ...
  },
  ...
}
```

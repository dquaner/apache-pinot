# JSON Index

JSON 索引可以被应用到 JSON 字符串列来加速该列的值查找和过滤。

## 什么时候适合使用 JSON 索引

可以使用 JSON 字符串来表示数组，map，嵌套字段，而不需要强制一个固定的模式。JSON 字符串非常灵活，但灵活性也带来了成本 — 在 JSON 字符串列上执行过滤代价很高。

假设我们在 `person` 列中存储了如下记录：

```javascript
{
  "name": "adam",
  "age": 30,
  "country": "us",
  "addresses":
  [
    {
      "number" : 112,
      "street" : "main st",
      "country" : "us"
    },
    {
      "number" : 2,
      "street" : "second st",
      "country" : "us"
    },
    {
      "number" : 3,
      "street" : "third st",
      "country" : "ca"
    }
  ]
}
```

如果不添加索引，想要基于 value 寻找一个 key 并过滤记录，我们需要从每个记录的 JSON 字符串中扫描并构建 JSON 对象，查找键，然后比较值。

例如，为了查找所有名字为 "adam" 的人，会使用下面这样的查询语句：

```sql
SELECT * 
FROM mytable 
WHERE JSON_EXTRACT_SCALAR(person, '$.name', 'STRING') = 'adam'
```

设计 JSON 索引是为了避免扫描和构建所有 JSON 对象，以加速 JSON 字符串列上的过滤。

## 配置 JSON 索引

为了启用 JSON 索引，需要在表配置中做如下配置：

### 版本 `0.12.0` 之后的配置

```javascript
{
  "tableIndexConfig": {
    "jsonIndexConfigs": {
      "person": {
        "maxLevels": 2,
        "excludeArray": false,
        "disableCrossArrayUnnest": true,
        "includePaths": null,
        "excludePaths": null,
        "excludeFields": null
      },
      ...
    },
    ...
  }
}
```

配置关键字 | 介绍  | 类型 | 默认值 
--- | --- | --- | ---
| **maxLevels**               | Max levels to flatten the json object (array is also counted as one level)                                                                                                                                                             | int          | -1 (不限制)                                       |
| **excludeArray**            | Whether to exclude array when flattening the object                                                                                                                                                                                    | boolean      | false (include array)                                |
| **disableCrossArrayUnnest** | Whether to not unnest multiple arrays (unique combination of all elements)                                                                                                                                                             | boolean      | false (calculate unique combination of all elements) |
| **includePaths**            | Only include the given paths, e.g. "_$.a.b_", "_$.a.c\[\*]_" (mutual exclusive with **excludePaths**). Paths under the included paths will be included, e.g. "_$.a.b.c_" will be included when "_$.a.b_" is configured to be included. | Set\<String> | null (include all paths)                             |
| **excludePaths**            | Exclude the given paths, e.g. "_$.a.b_", "_$.a.c\[\*]_" (mutual exclusive with **includePaths**). Paths under the excluded paths will also be excluded, e.g. "_$.a.b.c_" will be excluded when "_$.a.b_" is configured to be excluded. | Set\<String> | null (include all paths)                             |
| **excludeFields**           | Exclude the given fields, e.g. "_b_", "_c_", even if it is under the included paths.                                                                                                                                                   | Set\<String> | null (include all fields)                            |

#### Example:

With the following JSON document:

```json
{
  "name": "adam",
  "age": 20,
  "addresses": [
    {
      "country": "us",
      "street": "main st",
      "number": 1
    },
    {
      "country": "ca",
      "street": "second st",
      "number": 2
    }
  ],
  "skills": [
    "english",
    "programming"
  ]
}
```

With the default setting, we will flatten the document into the following records:

```json
{
  "name": "adam",
  "age": 20,
  "addresses[0].country": "us",
  "addresses[0].street": "main st",
  "addresses[0].number": 1,
  "skills[0]": "english"
},
{
  "name": "adam",
  "age": 20,
  "addresses[0].country": "us",
  "addresses[0].street": "main st",
  "addresses[0].number": 1,
  "skills[1]": "programming"
},
{
  "name": "adam",
  "age": 20,
  "addresses[1].country": "ca",
  "addresses[1].street": "second st",
  "addresses[1].number": 2,
  "skills[0]": "english"
},
{
  "name": "adam",
  "age": 20,
  "addresses[1].country": "ca",
  "addresses[1].street": "second st",
  "addresses[1].number": 2,
  "skills[1]": "programming"
}
```

With **maxLevels** set to 1:

```json
{
  "name": "adam",
  "age": 20
}
```

With **maxLevels** set to 2:

```json
{
  "name": "adam",
  "age": 20,
  "skills[0]": "english"
},
{
  "name": "adam",
  "age": 20,
  "skills[1]": "programming"
}
```

With **excludeArray** set to true:

```json
{
  "name": "adam",
  "age": 20
}
```

With **disableCrossArrayUnnest** set to true:

```json
{
  "name": "adam",
  "age": 20,
  "addresses[0].country": "us",
  "addresses[0].street": "main st",
  "addresses[0].number": 1
},
{
  "name": "adam",
  "age": 20,
  "addresses[0].country": "us",
  "addresses[0].street": "main st",
  "addresses[0].number": 1
},
{
  "name": "adam",
  "age": 20,
  "skills[0]": "english"
},
{
  "name": "adam",
  "age": 20,
  "skills[1]": "programming"
}
```

With **includePaths** set to \["$.name", "$.addresses\[\*].country"]:

```json
{
  "name": "adam",
  "addresses[0].country": "us"
},
{
  "name": "adam",
  "addresses[1].country": "ca"
}
```

With **excludePaths** set to \["$.age", "$.addresses\[\*].number"]:

```json
{
  "name": "adam",
  "addresses[0].country": "us",
  "addresses[0].street": "main st",
  "skills[0]": "english"
},
{
  "name": "adam",
  "addresses[0].country": "us",
  "addresses[0].street": "main st",
  "skills[1]": "programming"
},
{
  "name": "adam",
  "addresses[1].country": "ca",
  "addresses[1].street": "second st",
  "skills[0]": "english"
},
{
  "name": "adam",
  "addresses[1].country": "ca",
  "addresses[1].street": "second st",
  "skills[1]": "programming"
}
```

With **excludeFields** set to \["age", "street"]:

```json
{
  "name": "adam",
  "addresses[0].country": "us",
  "addresses[0].number": 1,
  "skills[0]": "english"
},
{
  "name": "adam",
  "addresses[0].country": "us",
  "addresses[0].number": 1,
  "skills[1]": "programming"
},
{
  "name": "adam",
  "addresses[1].country": "ca",
  "addresses[1].number": 2,
  "skills[0]": "english"
},
{
  "name": "adam",
  "addresses[1].country": "ca",
  "addresses[1].number": 2,
  "skills[1]": "programming"
}
```

### Legacy config before release `0.12.0`:

```javascript
{
  "tableIndexConfig": {        
    "jsonIndexColumns": [
      "person",
      ...
    ],
    ...
  }
}
```

旧的配置方式相当于新的配置方式中所有字段都设置为默认值。

注意 JSON 索引只能用在 `STRING/JSON` 列上，且该列的值为 JSON 字符串。

{% hint style="info" %}
当你在某一列上使用 JSON 索引时，建议将该列添加到 `noDictionaryColumns` 下以减少不必要的存储开销。

该配置属性的说明，请查看 [Raw value forward index](forward-index.md#yuan-shi-zhi-zheng-pai-suo-yin) 文档。
{% endhint %}

## 如何使用 JSON 索引

JSON 索引可以通过 `JSON_MATCH` 谓词使用：`JSON_MATCH(<column>, '<filterExpression>')` 。例如，查找名字为 "adam" 的所有人，查询语句如下：

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.name"=''adam''')
```

注意：过滤表达式中的引号应该进行转义。

{% hint style="info" %}
版本 `0.7.1` 中，使用老 `filterExpression` 的语法：`'name=''adam'''`
{% endhint %}

## 支持的过滤表达式

### 简单的关键字查找

Find all persons whose name is "adam":

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.name"=''adam''')
```

{% hint style="info" %}
In release `0.7.1`, we use the old syntax for filterExpression: `'name=''adam'''`
{% endhint %}

### 链式关键字查找

Find all persons who have an address (one of the addresses) with number 112:

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.addresses[*].number"=112')
```

{% hint style="info" %}
In release `0.7.1`, we use the old syntax for filterExpression: `'addresses.number=112'`
{% endhint %}

### 嵌套过滤表达式

Find all persons whose name is "adam" and also have an address (one of the addresses) with number 112:

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.name"=''adam'' AND "$.addresses[*].number"=112')
```

{% hint style="info" %}
In release `0.7.1`, we use the old syntax for filterExpression: `'name=''adam'' AND addresses.number=112'`
{% endhint %}

### 访问数组

Find all persons whose first address has number 112:

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.addresses[0].number"=112')
```

{% hint style="info" %}
In release `0.7.1`, we use the old syntax for filterExpression: `'"addresses[0].number"=112'`
{% endhint %}

### 检查字段是否存在

Find all persons who have a phone field within the JSON:

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.phone" IS NOT NULL')
```

{% hint style="info" %}
In release `0.7.1`, we use the old syntax for filterExpression: `'phone IS NOT NULL'`
{% endhint %}

Find all persons whose first address does not contain floor field within the JSON:

```sql
SELECT ... 
FROM mytable
WHERE JSON_MATCH(person, '"$.addresses[0].floor" IS NULL')
```

{% hint style="info" %}
In release `0.7.1`, we use the old syntax for filterExpression: `'"addresses[0].floor" IS NULL'`
{% endhint %}

## JSON 上下文是被维护的

The JSON context is maintained for object elements within an array, i.e. the filter won't cross-match different objects in the array.

To find all persons who live on "main st" in "ca":

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.addresses[*].street"=''main st'' AND "$.addresses[*].country"=''ca''')
```

This query won't match "adam" because none of his addresses matches both the street and the country.

If JSON context is not desired, use multiple separate `JSON_MATCH` predicates. E.g. to find all persons who have addresses on "main st" and have addressed in "ca" (doesn't have to be the same address):

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.addresses[*].street"=''main st''') AND JSON_MATCH(person, '"$.addresses[*].country"=''ca''')
```

This query will match "adam" because one of his addresses matches the street and another one matches the country.

Note that the array index is maintained as a separate entry within the element, so in order to query different elements within an array, multiple `JSON_MATCH` predicates are required. E.g. to find all persons who have first address on "main st" and second address on "second st":

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(person, '"$.addresses[0].street"=''main st''') AND JSON_MATCH(person, '"$.addresses[1].street"=''second st''')
```

## 支持的 JSON 值

### Object

查看上面的示例。

### Array

```javascript
["item1", "item2", "item3"]
```

To find the records with array element "item1" in "arrayCol":

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(arrayCol, '"$[*]"=''item1''')
```

To find the records with second array element "item2" in "arrayCol":

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(arrayCol, '"$[1]"=''item2''')
```

### Value

```javascript
123
1.23
"Hello World"
```

To find the records with value 123 in "valueCol":

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(valueCol, '"$"=123')
```

### Null

```javascript
null
```

To find the records with null in "nullableCol":

```sql
SELECT ... 
FROM mytable 
WHERE JSON_MATCH(nullableCol, '"$" IS NULL')
```

{% hint style="warning" %}
版本 `0.7.1` 中，JSON 字符串必须为对象（不能为 `null`, Value, or Array）；不支持多维数组。
{% endhint %}

## 限制

1. The key (left-hand side) of the filter expression must be the leaf level of the JSON object, e.g. `"$.addresses[*]"='main st'` won't work.
过滤器表达式的 key（左边）必须是 JSON 对象的叶子层，例如。`"$.addresses[*]"='main st'` 无效。
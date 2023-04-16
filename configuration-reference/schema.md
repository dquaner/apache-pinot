# Schema

模式用于定义一张 Pinot 表中的各个列的列名，数据类型，以及其它列的信息。

Pinot 模式包括以下字段：

字段 | 描述 
--- | ---
**schemaName**         | 定义模式名，通常和表名相同。一个混合表的离线表和实时表应该使用相同的模式。
**dimensionFieldSpec** | 为每一个维度列定义一个 dimensionFieldSpec 字段，详细信息查看下面的 [DimensionFieldSpec](schema.md#dimensionfieldspec) 小节。
**metricFieldSpec**    | 为每一个指标列定义一个 metricFieldSpec 字段，详细信息查看下面的 [MetricFieldSpec](schema.md#metricfieldspec) 小节。
**dateTimeFieldSpec**  | 为时间列定义一个 dateTimeFieldSpec 字段，可以有多个时间列，详细信息查看下面的 [DateTimeFieldSpec](schema.md#datetimefieldspec) 小节。

下面是每一种字段的详细介绍：

### DimensionFieldSpec

为每一个维度列定义一个 dimensionFieldSpec 字段，dimensionFieldSpec 中的属性包括：

属性 | 描述 
--- | ---
name             | Name of the dimension column. 
dataType         | Data type of the dimension column. Can be INT, LONG, FLOAT, DOUBLE, BOOLEAN, TIMESTAMP, STRING, BYTES.
defaultNullValue | Represents null values in the data, since Pinot doesn't support storing null column values natively (as part of its on-disk storage format). If not specified, an internal default null value is used as listed here.
singleValueField | Boolean indicating if this is a single-valued or a multi-valued column. Multi-valued column is modeled as a list, where the order of the values are preserved and duplicate values are allowed. Individual rows don’t necessarily have the same number of values. Typical use case for this would be a column such as `skillSet` for a person (one row in the table) that can have multiple values such as `Real Estate, Mortgages`. The default null value for a multi-valued column is a single `defaultNullValue`, e.g. `[Integer.MIN_VALUE]`. |

#### Internal default null values for dimension

| Data Type | Internal Default Null Value                                                                                       |
| --------- | ---
| INT       | ​[Integer.MIN\_VALUE](https://docs.oracle.com/javase/7/docs/api/java/lang/Integer.html#MIN\_VALUE)​               |
| LONG      | ​[Long.MIN\_VALUE](https://docs.oracle.com/javase/7/docs/api/java/lang/Long.html#MIN\_VALUE)​                     |
| FLOAT     | ​[Float.NEGATIVE\_INFINITY](https://docs.oracle.com/javase/7/docs/api/java/lang/Float.html#NEGATIVE\_INFINITY)​   |
| DOUBLE    | ​[Double.NEGATIVE\_INFINITY](https://docs.oracle.com/javase/7/docs/api/java/lang/Double.html#NEGATIVE\_INFINITY)​ |
| BOOLEAN   | 0 (`false`)                                                                                                       |
| TIMESTAMP | 0 (`1970-01-01 00:00:00 UTC`)                                                                                     |
| STRING    | "null"                                                                                                            |
| BYTES     | byte array of length 0                                                                                            |

### MetricFieldSpec

A metricFieldSpec is defined for each metric column. Here's a list of fields in the metricFieldSpec
为每一个指标列定义一个 metricFieldSpec 字段，metricFieldSpec 中的属性包括：

属性 | 描述 
--- | ---
name             | Name of the metric column                                                                                                                                                                               |
dataType         | Data type of the column. Can be INT, LONG, FLOAT, DOUBLE, BIG\_DECIMAL, BYTES (for specialized representations such as HLL, TDigest, etc, where the column stores byte serialized version of the value) |
defaultNullValue | Represents null values in the data. If not specified, an internal default null value is used, as listed here.                                                                                           |

#### Internal default null values for metric

| Data Type    | Internal Default Null Value |
| ------------ | --------------------------- |
| INT          | 0                           |
| LONG         | 0                           |
| FLOAT        | 0.0                         |
| DOUBLE       | 0.0                         |
| BIG\_DECIMAL | 0.0                         |
| BYTES        | byte array of length 0      |

### DateTimeFieldSpec

A dateTimeFieldSpec is used to define time columns of the table. Here's a list of the fields in a dateTimeFieldSpec

| Property         | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| name             | Name of the date time column                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| dataType         | Data type of the date time column. Can be STRING, INT, LONG or TIMESTAMP                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| format           | <p>The format of the time column. The syntax of the format is <code>timeSize:timeUnit:timeFormat</code></p><p>timeFormat can be either EPOCH or SIMPLE_DATE_FORMAT. If it is SIMPLE_DATE_FORMAT, the pattern string is also specified. For example:</p><p>1:MILLISECONDS:EPOCH - epoch millis</p><p>1:HOURS:EPOCH - epoch hours</p><p>1:DAYS:SIMPLE_DATE_FORMAT:yyyyMMdd - date specified like <code>20191018</code></p><p>1:HOURS:SIMPLE_DATE_FORMAT:EEE MMM dd HH:mm:ss ZZZ yyyy - date specified like <code>Mon Aug 24 12:36:50 America/Los_Angeles 2019</code></p><p>Noted that if TIMESTAMP type is used in dataType, format is ignored because JDBC standard SQL requires TIMESTAMP in <code>yyyy-[m]m-[d]d hh:mm:ss[.f...]</code> format.</p> |
| granularity      | <p>The granularity in which the column is bucketed. The syntax of granularity is<br><code>bucket size:bucket unit</code><br>For example, the format can be milliseconds <code>1:MILLISECONDS:EPOCH</code>, but bucketed to 15 minutes i.e. we only have one value for every 15 minute interval, in which case granularity can be specified as <code>15:MINUTES</code></p>                                                                                                                                                                                                                                                                                                                                                                            |
| defaultNullValue | <p>Represents null values in the data. If not specified, an internal default null value is used. The internal default null value is the same as dimension field.</p><p></p><p><strong>For the main time column of the table (time column specified in the <code>segmentsConfig</code></strong></p><p> <strong>in the table config), the main time column value must be in the range of <code>1971-01-01 UTC</code> to <code>2071-01-01 UTC</code> for segment management purpose (e.g. retention, time boundary). If the specified default null value is not within this range, segment creation time is used.</strong></p>                                                                                                                          |

### Advanced fields

Apart from these, there's some advanced fields. These are common to all field specs.

| Property              | Description               |
| --------------------- | ------------------------- |
| maxLength             | Max length of this column |
| virtualColumnProvider | Column value provider     |
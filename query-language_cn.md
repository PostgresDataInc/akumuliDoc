---
description: Akumuli查询语言参考手册。
---

# 查询语言

Akumuli使用REST接口进行数据检索，数据查询使用 [**api/query**](api-endpoints_cn.md#读取查询) 端点，元数据查询使用 [**api/search**](api-endpoints_cn.md#搜索) 端点。

## 数据模型

所有数据使用不同量度\(metric\)分割，可以把量度认为是若干序列名的命名空间，每个序列名以量度名开始。

```text
cpu.user host=pg-balancer OS=Trusty arch=x86_amd64 region=NE
```

这里，**cpu.user** 是一个量度名，**host=pg-balancer** 是一个“标签/值”对，Akumuli的序列名搜索使用 [布尔模型 \(BIR\)](https://en.wikipedia.org/wiki/Standard_Boolean_model)，每个查询应该指定一个度量以及一组可选的“标签/值”对\(使用[条件字段](query-language_cn.md#条件字段)\)。具有此度量名并包含此标签/值对的序列将添加到结果集中，如果查询没有where字段，它将返回具有相同度量名的所有时序。

每个序列名都连接到数据点列表(实际的时序数据)，每个数据点包含时间戳\(64位，纳秒精度\)和值\(双精度浮点\)，需要在查询中指定数据的搜索范围。

## 查询对象

为了检索数据，应该创建一个查询对象，该对象描述需要什么数据，以及应该具有的特性。查询以JSON编码，并发送到 [查询端点](api-endpoints_cn.md#读取查询)。

## 错误处理

查询解析错误使用 [RESP协议](writing-data_cn.md#错误信息) 反馈，响应中的唯一一行将以“-”开始，后跟错误消息。

某些错误可以使用HTTP错误码反馈\(例如，使用错误的API端点时\)，查询解析错误使用错误消息反馈，而查询处理错误通常使用HTTP错误代码反馈。

## 查询类型

查询对象可以是下列类型之一：

* [选择查询](query-language_cn.md#选择查询)
* [聚合查询](query-language_cn.md#聚合查询)
* [分组集计查询](query-language_cn.md#分组集计查询)
* [联接查询](query-language_cn.md#联接查询)

### 选择查询

这是最简单的查询类型，返回未经聚合的原始时序数据，可以有一个后处理步骤\(例如，速率或滑动窗口计算\)。 

选择查询可以返回多个序列，但它们必须具有相同的度量名。

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| [select](query-language.md#选择字段) | 是 | 度量名 |
| [range](query-language.md#范围字段) | 是 | 时间范围 |
| [where](query-language.md#条件字段) | 否 | 标签过滤器 |
| [group-by](query-language.md#分组字段) | 否 | 序列转换 |
| [order-by](query-language.md#排序字段) | 否 | 结果集中数据点的顺序 |
| [filter](query-language.md#过滤器字段) | 否 | 基于值过滤 |
| [limit](query-language.md#限量和偏移字段) | 否 | 限制输出的大小 |
| [offset](query-language.md#限量和偏移字段) | 否 | 查询输出的偏移量 |

### 聚合查询

此查询可用于计算时间序列上的聚合，对于每个时间序列，查询只返回一个结果。

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| [aggregate](query-language.md#聚合字段) | 是 | 度量名和聚合函数 |
| [range](query-language.md#范围字段) | 是 | 时间范围 |
| [group-by](query-language.md#分组字段) | 否 | 序列转换 |
| [where](query-language.md#条件字段) | 否 | 标签过滤器 |
| [output](query-language.md#输出字段) | 否 | 设置输出格式 |

### 分组聚合查询

此查询用于对时序数据进行降低采样，它将所有的数据点划分成一系列大小相等的容器，每个容器计算一个值。可以在 [聚合查询](query-language_cn.md#聚合查询) 使用的聚合函数同样可以用于分组聚合。聚合查询和分组聚合查询的不同之处在于，聚合对于每个序列只产生一个值，而分组聚合可以生成一个固定步长的时序。此外，可以在分组聚合查询中使用多个聚合函数。

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| [group-aggregate](query-language.md#分组聚合字段) | 是 | Query specific parameters |
| [range](query-language.md#range-field) | 是 | 时间范围 |
| [where](query-language.md#where-field) | 否 | 标签过滤器 |
| [group-by](query-language.md#分组字段) | 否 | 序列转换 |
| [order-by](query-language.md#排序字段) | 否 | 结果集中数据点的顺序 |
| [filter](query-language.md#过滤器字段) | 否 | 值过滤器 |
| [limit](query-language.md#限量和偏移字段) | 否 | 限制输出的大小 |
| [offset](query-language.md#限量和偏移字段) | 否 | 查询输出的偏移量 |

### 联接查询

联接查询可用于将多个度量对齐，将具有相同标签度量名不同的序列组合在一起。输出结果使用 [批量加载格式](writing-data.md#批量写入度量数据)，每个序列的序列名以 [复合序列名格式](writing-data_cn.md#复合序列名) 连接在一起。

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| [join](query-language.md#join-field) | Yes | 联接的度量列表 |
| [range](query-language.md#range-field) | Yes | 时间范围 |
| [where](query-language.md#where-field) | No | 标签过滤器 |
| [group-by](query-language.md#分组字段) | No | 序列转换 |
| [order-by](query-language.md#排序字段) | No | 结果集中数据点的顺序 |
| [filter](query-language.md#过滤器字段) | No | 值过滤器 |
| [limit](query-language.md#限量和偏移字段) | No | 限制输出的大小 |
| [offset](query-language.md#限量和偏移字段) | No | 查询输出的偏移量 |

## 查询字段

查询对象以JSON编码，包含预定义的字段集，其中一些字段是必须的，其他字段是可选的。

### 范围字段

范围字段表示查询获取数据的时间间隔。

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| "range" | { "from": "20180530T123000", "to": "20180530T130000" } | 字段应该包含带有两个键 “from” 和 “to” 的字典，它们的值都是时间戳。 |

这两个时间戳都应该使用基本的ISO 8601格式进行编码，RESP协议的数据采集也使用这种格式。数据查询必须使用 “range.from” 和 “range.to” 字段，如果 “range.from” 小于 “range.to”，将按升顺\(从旧到新\)返回时序数据点；如果 “range.from” 大于 “range.to ”，则按降序\(从新到旧\)返回时序数据点。

### 选择字段

select字段用于告诉Akumuli应该获取哪个度量。

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| "select" | "metric.name" | 度量名 |

此字段定义查询类型，如果使用此字段，查询将是一个简单的 [选择查询](query-language_cn.md#选择查询)。只能有一个“select”字段，并且此字段只能有一个度量名。查询将获取具有此度量名的所有序列，具体系列可以通过“where”字段进行进一步筛选。

### 聚合字段

聚合字段是创建 [聚合查询](query-language_cn.md#聚合查询) 所必需的，字段类型是具有以下格式的字典：

```text
"aggregate": { <metric-name>: <aggregation-function> }
```

只能有一个度量名和聚合函数对，可用的聚合函数如下：

| 名称 | 描述 |
| --- | --- |
| count | 序列中的元素数量 |
| max | 序列中的最大元素 |
| min | 序列中的最小元素 |
| mean | 平均值 |
| sum | 序列中所有值的合计 |
| min\_timestamp | 最小元素的时间戳 |
| max\_timestamp | 最大元素的时间戳 |

聚合查询只计算指定 [时间范围](query-language.md#范围字段) 内的值聚合，如果范围内没有值，查询将返回错误。

### 分组聚合字段

组聚合字段是进行[分组聚合查询](query-language_cn.md#分组聚合查询)所必需的，该字段是具有以下格式的字典：

```text
{
    "group-aggregate": {
        "metric": <metric-name>,
        "step": <time-duration>,
        "func": <function-name>
}
```

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| group-aggregate.metric | String | 度量名 \(与 [select](query-language_cn.md#选择字段) 相同\) |
| group-aggregate.step | String | 聚合步长 \(10s、1h、5m\) |
| group-aggregate.func | String | 聚合函数 |
| group-aggregate.func | List | 聚合函数列表 |

#### 使用单个函数

如果在`分组聚合` 字段中只使用一个聚合函数，输出具有以下格式：

```text
+cpu:min host=host1\r\n
+20170101T221015\r\n
+0.05\r\n
```

原始序列的序列名发生了变化，标签保持不变，但度量名有一个 :<函数名> 后缀。在上面的示例中，原始的序列名是 ‘cpu host=host1’，而最终的序列名是 ‘cpu:min host=host1’。 

#### 使用函数列表

如果在分组聚合字段中使用多个聚合函数，输出将具有以下格式：

```text
+cpu:min|cpu:max host=host1\r\n
+20170101T221015\r\n
*2\r\n
+0.05\r\n
+99.7\r\n
```

如前所述，度量名同样发生变化，而且使用的是 [复合序列名格式](writing-data_cn.md#复合序列名)。查询将为列表中的每个聚合函数返回一个序列，这些序列拥有相同的时间戳，但值不同\(因为生成函数不同\)。然后，将这些序列连接在一起，并将它们以 [批量格式](writing-data.md#批量写入度量数据) 返回。

### 联接字段

联接字段用于进行联接查询，此字段的类型为列表。列表应包含有效的度量名，示例：

```text
{
    "join": ["cpu", "mem", "iops"]
}
```

这里， `cpu`、 `mem`、以及 `iops` 是不同的度量名。查询处理器将在该度量中找到具有相同标签集的序列名，并将它们联接起来。例如，如果我们有三个序列 “cpu host=host1”、“mem host=host1” 和 “iops host=host1”，它们将合并在一起，生成单个序列 “cpu\|mem\|iops host=host1”，输出将返回 [批量格式](writing-data.md#批量写入度量数据) 的记录。

```text
+cpu|mem|iops host=host1\r\n
+20161231T235500\r\n
*3\r\n
+10.5\r\n
+4870\r\n
+148\r\n
```

### 条件字段

where字段用于限制查询返回的序列数。

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| "where" | { "tag-name": "tag-value" } | 仅包括标签 "tag-name" 的值设置为 "tag-value" 的序列。 |
| "where" | { "tag-name": \[ "value1", "value2" \] } | 仅包括标签 "tag-name" 的值设置为 "value1"或者"value2" 的序列。 |

可以在一个where字段中指定多个标签，连同度量名\(可以多个\)一起用于在索引内[搜索序列](query-language_cn.md#数据模型)。

### 分组字段

分组字段用于将多个序列合并在一起，如果使用 `group-by` 字段指定标签名称，则所有拥有此标签并具有相同标签值的序列都将合并为一个。这些序列的所有数据点将合并在一起，由此产生的时序将包含原始序列的所有数据点，序列名称将只包含在 `group-by` 字段中指定的那些标签。

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| "group-by" | \[ "tag1", "tag2", ..., "tagN" \] | 结果序列名应该具有的标签列表。 |

假设需要存储阀门压力测量数据，每个阀门中的压力是由两个独立的传感器测量的，因此您最终得到的模式是：`pressure_kPa valve_num=XXX sensor_num=YYY`，度量 `pressure_kPa` 有两个标签：`valve_num` 和 `sensor_num`。如果查询这个序列，将得到以下结果\(省略_\r\n_\)：

```text
+pressure_kPa valve_num=0 sensor_num=0
+20160118T171000.000000000
+204.0
+pressure_kPa valve_num=0 sensor_num=1
+20160118T171000.000000000
+204.1
+pressure_kPa valve_num=1 sensor_num=0
+20160118T171000.000000000
+208.0
+pressure_kPa valve_num=1 sensor_num=1
+20160118T171000.000000000
+208.2
...
```

传感器和阀门的每一种组合都会产生自己的时序数据，如果只想通过阀门对数据进行分组，可以使用“group-by”字段。在查询中添加  `"group-by": [ "valve_num" ]`  字段，结果将如下所示：

```text
+pressure_kPa valve_num=0
+20160118T171000.000000000
+204.0
+pressure_kPa valve_num=0
+20160118T171000.000000000
+204.1
+pressure_kPa valve_num=1
+20160118T171000.000000000
+208.0
+pressure_kPa valve_num=1
+20160118T171000.000000000
+208.2
...
```

### 排序字段

此字段可用于控制查询输出的数据点顺序。

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| "order-by" | "series" | 以序列名将输出排序 |
| "order-by" | "time" | 以时间戳将输出排序 |

此字段接受单个字符串，可以是“series”或“time”。如果 `order-by` 是“series”，则结果将首先按序列名称序，然后再按时间戳排序。如果 `order-by` 是“time”，那么数据点将首先按时间戳排序，然后再按序列名排序。

### 输出字段

此字段用于控制输出格式。

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| "output" | { "format": "csv", "timestamp": "raw" } | 设置输出格式为 "csv"、时间戳格式为 "raw"。 |
| "output" | { "format": "resp", "timestamp": "iso" } | 设置输出格式为 "resp"、时间戳格式为 "iso"。 |

此字段具有两个可设置值的字典，第一个是 `output.format`，它可以设置为“resp”或“csv”。默认使用第一个值，使用 [RESP序列化](writing-data.md#序列化) 将输出格式化，这与发送数据给Akumuli的格式相同。第二个值将输出格式设置为CSV，查询将 `output.format` 设置为“csv”的输出看起来类似这样：

```text
test tag=Foo, 20160118T173724.646397000, 999996
test tag=Foo, 20160118T173724.647397000, 999997
test tag=Foo, 20160118T173724.648397000, 999998
test tag=Foo, 20160118T173724.649397000, 999999
```

第二个是 `output.timestamp`，它控制查询输出中时间戳的格式。如果设置为“raw”，Akumuli将时间戳格式化为64位整数。

```text
test tag=Foo, 1453127844646397000, 999996
test tag=Foo, 1453127844647397000, 999997
test tag=Foo, 1453127844648397000, 999998
test tag=Foo, 1453127844649397000, 999999
```

如果设置为“iso”，时间戳将按照ISO8601标准格式化。

### 过滤器字段

过滤器字段用于以值过滤数据点。

| 字段 | 格式 | 描述 |
| --- | --- | --- |
| "format" | { "gt": 10, "lt": 100 } | 过滤掉所有小于等于10和大于等于100的值。 |
| "format" | { "ge": 0, "le": 1 } | 过滤掉所有负数值和所有大于1的值。 |

此字段包含一个带有谓词的字典，可支持的谓词包括："gt" \(大于\)、"ge" \(大于等于\)、"lt" \(小于\)、以及"le" \(小于等于\)。如果想读取符合某个范围的值，可以组合两个谓词，例如：`"filter: {"gt": 0, "lt": 10 }` 选择0和10之间但不包括0和10的所有值。如果需要，可以只使用谓词。

如果值的返回量较少，过滤器字段的使用可以加快查询的执行速度。在这种情况下，查询引擎不必读取磁盘上的所有数据，只需读取满足查询所需数据的页面即可。

#### 多维过滤器

过滤器字段可以与联接查询一起使用。如果是这种情况，必须指定过滤器应用的度量。

```text
{
    "join": ["cpu", "mem", "iops"],
    "filter": {
        "cpu": { "gt": 200 },
        "mem": { "lt": 100 }
    },
    ...
}
```

这个例子里，过滤器 &gt;200 将应用于度量 “cpu”，过滤器 &lt;100 将应用于度量 “mem”。

### 限量和偏移字段

可以使 `limit` 和 `offset` 查询字段来限制返回元组的数目，并在查询输出开始时跳过部分元组，这与SQL中的LIMIT和OFFSET子句含义一样。

\(chunk是突然出现在这里的一个词，直译为大块，无上下文参照无法理解其具体含义，猜测是其存储引擎上的一个单位\)如果需要读取大块中的所有数据，不要使用此字段。Akumuli惰性执行查询，若要读取大块中的数据，发出正常查询\(没有限量和偏移量\)，读取第一个大块\(无需断开服务器\)，当完成第一个大块时可以继续读取下一个，如此反复。查询将按照通过TCP连接读取数据的方式执行，当处理数据而停止读取时，服务器上的查询执行将暂停，继续读取时恢复执行。


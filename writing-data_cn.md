# 写入数据

Akumuli协议基于[Redis协议](http://redis.io/topics/protocol)\(又名RESP或Redis序列化协议)，该协议设计成易于实现、高可读性、并且解析速度快。除了RESP之外，Akumuli还支持[OpenTSDB telnet风格API](http://opentsdb.net/docs/build/html/api_telnet/put.html)。

## 序列化

Akumuli借用RESP序列化格式，但不借用协议。序列化格式使用五种数据类型：

* 字符串
* 整数
* 错误信息
* 数组
* 批量串\(Bulk string\)

### 字符串

简单字符串以加号开头，后跟字符串的字符，以CRLF\(回车和换行符\)结尾，或仅以换行符结尾。字符串的长度限制为1k字节，不能包含换行或回车字符。

```text
+proc.net.bytes\r\n
```

字符串可以用来表示时间戳、浮点值、或者序列名。

### 整数

整数以冒号开头，后跟一系列数字，以CRLF结束。

```text
:314159\r\n
```

Akumuli中的整数值限制为84位数字，用于表示数值或时间戳。

### 错误

错误以减号开头，后跟包含错误消息的若干字符，以CRLF结束。

```text
-RESP Error:  declared object size is too large
```

此数据类型用于向客户端返回有关错误的信息。

### 数组

数组是一种复合数据结构，可以由任意数量的不同类型的值组成。

* 数组以星号开头，后跟数组中的元素数，以CRLF结尾。
* 每个元素是一个RESP编码的值

```text
*2\r\n
:1\r\n
+second\r\n
```

### 批量串

批量串用于表示任意大的\(最多1MB\)二进制对象。

* 值以美元符开始，后跟字符串的长度，以CRLF(或一个换行符)结束。
* 实际的批量串数据以CRLF结束，字符串的长度应该与$字符后面的值匹配。

```text
$11\r\n
Hello world\r\n
```

## 写入单个度量数据

单独发送每个数据点，这是将数据写入Akumuli实例的最简单方法，每个数据点包含以下内容：

* 序列名
* 时间戳
* 值

### 序列名

序列名称具有以下格式：: `<量度名> <标签>`，其中`<标签>`是由空格分隔的键值对列表。必须同时指定度量名和标签\(至少有一个键值对\)，否则它不是有效的序列名。应该使用[RESP字符串](writing-data_cn.md#字符串)方式编码。

示例：

```text
+cpu_user host=hostname region=NW\r\n
```

这里，"cpu\_user"是度量名，而"host"和"region"是标签。

```text
+network.loadavg host=postgres\r\n
```

度量名为"network.loadavg"，标签"host"的值设置为"postgres"，可以在[数据模型](data-model_cn.md)章节查阅更多关于序列名的信息。

### 时间戳

时间戳必须是UTC时间\(Akumuli不支持使用本地时间\)，应该使用RESP [字符串](writing-data_cn.md#字符串) 或 [整数](writing-data.md#整数) 方式对时间戳编码。

如果使用**字符串**，Akumuli尝试将其解释为ISO 8601编码的UTC时间日期\(精度为纳秒或更低\)。

```text
+20141210T074343.999999999\r\n
```

注意：只支持基本的ISO 8601格式，所有时间戳都应该是UTC时间戳，不支持时区。

如果使用整数，其值解释为纳秒精度的64位时间戳，纪元开始于1970年1月1日\(Unix纪元\)。If the **integer** was used the value will be interpreted as a 64-bit timestamp with nanosecond precision. The beginning of the epoch is Jan-1 1970 \(Unix epoch\).

```text
:1418224205000000000\r\n
```

### 值

值以RESP [字符串 ](writing-data_cn.md#字符串)或者[整数](writing-data_cn.md#整数) 格式编码，如果使用字符串，它解释为以字符串表示的浮点值，并且支持科学记数法。

```text
+3.14159\r\n
```

注意，浮点值的字符串表示形式可能会降低精度，为了不丢失精度，可以使用下列方法打印浮点数：

```text
printf("%.17g", float64);
```

如果使用**整数**，它的值正如看到的那样。注意，由于Akumuli使用由IEEE 754标准定义的双精度浮点数，只能精确地表示54位整数。

### 撰写消息

每个单独的消息以序列名开始，后跟时间戳和值。

完整的消息看起来类似这样\(\r\n替换为实际的换行\)：

* 字符串时间戳、整数值和带有两个键的字符串id：

  ```text
  +balancers.memusage host=machine1 region=NW
  +20141210T074343.999999999
  :31
  ```

* 整数时间戳、字符串值和带有一个键的字符串id：

  ```text
  +balancers.cpuload host=machine1 region=NW
  :1418224205000000000
  +22.0
  ```

**重要：** 即使是流的最后一行也必须以CRLF结束。

可以将多个消息连接在一起，通过TCP连接发送。Akumuli读取传入的数据流，对其进行分析并写入各个数据点。

## 错误信息

如果一切正常，Akumuli不会回复任何信息；如果出现任何错误，它将返回错误消息。客户端可以从套接字异步读取数据\(非强制\)来接收错误消息。错误消息是RESP编码，以“-”开头。

## 工作流

为了发送数据给Akumuli，客户端需要打开TCP连接然后开始数据发送。作为可选项，客户端可以并行地从套接字读取数据，Akumui只在发生错误时才会返回某种信息。发生错误时，Akumuli发送错误消息后会关闭连接。

### 延迟写

目前，Akumuli只能写入有序数据，每个单独时序的数据点应该按时间戳排序\(只要保证每个时序的写入是有序的，不同时序的数据点之间可以乱序\)。

## 批量写入度量数据

批量传输可以减小传输尺寸，它允许将具有相同时间戳和来源的值组合在一起，从而最大限度地减少传输数据中的冗余。

```text
+mem.usage host=machine1 region=NW\r\n
+20180102T000200\r\n
+87.4\r\n
+cpu.user host=machine1 region=NW\r\n
+20180102T000200\r\n
+22.1\r\n
```

在本例中，_"mem.usage host=machine1 region=NW"_ 和 _"cpu.user host=machine1 region=NW"_  来自相同主机，它们共享标签集和时间戳\(因为收集器同时生成这两个度量，以及其他数百个类似的度量\)，这两个度量之间唯一的区别是度量名(cpu.user与mem.usage)和值。在这种情况下，合理的做法是批量发送，不必重复每个值的标签和时间戳\(特别是当我们有几十个值，而不仅仅是两个值时\)。如果使用批量格式，此示例看起来类似：

```text
+mem.usage|cpu.user host=machine1 region=NW\r\n
+20180102T000200\r\n
*2\r\n
+87.4\r\n
+22.1\r\n
```

要以批量格式向Akumuli写入数据，必须指定复合序列名、时间戳\(整数或字符\)和一个值数组\(每个值可以是整数或字符串\)。

### 复合序列名

复合序列名格式包含以管道符分隔的度量名列表和一组标签：

```text
<metric1>|<metric2>|...|<metricN> <tags>
```

复合序列名中的度量名列表不能包含任何空格，在存储端会转换为序列名列表：`<metric1> <tags>`, `<metric2> <tags> ……

```text
+cpu.real|cpu.user|cpu.sys host=machine1 region=NW
```

### 时间戳

序列名后跟时间戳，与[前边的描述](writing-data_cn.md#时间戳)使用相同方式。

### 值数组

值列表使用 [RESP数组](writing-data_cn.md#数组) 表示，RESP数组以星号开头，后跟数组中的元素数。这个数字必须与复合序列名中的度量数量匹配！后面必须有值\(值的数量也必须与度量名数量匹配，并且所有值的顺序也必须匹配，例如，如果cpu.real在复合序列名中排在第一位，那么它的数量值必须在数组中排在第一位\)。

示例：

```text
+cpu.real|cpu.user|cpu.sys host=machine1 region=NW
+20141210T074343
*3
+3.12
+8.11
+12.6
```

这将产生三次写：

* 序列名：cpu.real host=machine1 region=NW, TS: 20141210T074343, Value: 3.12
* 序列名：cpu.user host=machine1 region=NW, TS: 20141210T074343, Value: 8.11
* 序列名：cpu.sys host=machine1 region=NW, TS: 20141210T074343, Value: 12.6

## 字典模式

很多时候，客户端事先知道将要发送的序列，提供序列名字典将有助于减少传输数据冗余。考虑这个示例：

 In many situations client knows what series it will be sending in advance. This can help to cut down the redundancy in transferred data by providing the series name dictionary. Consider this example:

```text
+mem.usage host=machine1 region=NW\r\n
+20180102T000200\r\n
+87.4\r\n
+mem.usage host=machine1 region=NW\r\n
+20180102T000201\r\n
+87.5\r\n
+mem.usage host=machine1 region=NW\r\n
+20180102T000202\r\n
+88.1\r\n
```

这里，行之间发生变化的是时间戳和值，序列名总是相同的。客户端每次都需要发送序列名，每个数据点Akumuli不得不对它进行一次解析。为了将此开销降到最低，客户端可以发送序列名的预计算字典，此字典将序列名映射到用户定义的整数ID。然后，可以使用这个ID来代替协议中的序列名，这种方式也适用于BULK协议。

### 字典发送

字典只能在TCP会话开始时发送，使用[RESP数组](writing-data_cn.md#数组)表示。每个键值对使用数组的两个连续元素表示，序列名使用[RESP字符串](writing-data_cn.md#字符串)表示，序列名后跟使用[RESP整数](writing-data_cn.md#整数)表示的唯一ID。

```text
*4\r\n
+cpu.user host=machine1 region=NW\r\n
:1\r\n
+mem.usage host=machine1 region=NW\r\n
:2\r\n
```

这个字典稍后可以用于传输，若要使用字典中的序列，应该指定它的ID来代替序列名(或复合序列名)，ID应该以[RESP整数](writing-data_cn.md#整数)编码。

```text
:1\r\n
+20180102T000200\r\n
+11.2\r\n
:2\r\n
+20180102T000200\r\n
+43.99\r\n
```

这里，第一个数据点对应于序列 **cpu.user**，而第二个对应于 **mem.usage**。

可以使用一个数组或多个连续数组发送字典，唯一的要求是ID对于整个TCP会话应该是唯一的，并且应该首先发送，随后的数据点可以使用或忽略字典。

## OpenTSDB telnet风格API

Akumuli有限支持OpenTSDB telnet风格API，现在仅支持 `put` 命令，数据可以用以下格式插入：`put <metric-name> <timestamp> <value> <list-of-tags>`。示例：

```text
put cpu.user 1483228800 10.005344383927394 OS=Ubuntu_14.04 arch=x64 host=host_0 instance-type=m3.large rack=86 region=eu-central-1 team=NJ
put cpu.sys 1483228800 9.9992693762580025 OS=Ubuntu_14.04 arch=x64 host=host_0 instance-type=m3.large rack=86 region=eu-central-1 team=NJ
put cpu.real 1483228800 10.002083289000792 OS=Ubuntu_14.04 arch=x64 host=host_0 instance-type=m3.large rack=86 region=eu-central-1 team=NJ
put idle 1483228800 9.9815370970857522 OS=Ubuntu_14.04 arch=x64 host=host_0 instance-type=m3.large rack=86 region=eu-central-1 team=NJ
put mem.commit 1483228800 9 OS=Ubuntu_14.04 arch=x64 host=host_0 instance-type=m3.large rack=86 region=eu-central-1 team=NJ
put mem.virt 1483228800 10 OS=Ubuntu_14.04 arch=x64 host=host_0 instance-type=m3.large rack=86 region=eu-central-1 team=NJ
```

### 时间戳

OpenTSDB使用1秒精度的时间戳，这是常规Unix时间戳\(自纪元以来的秒数\)，Akumuli可以接受这种格式的时间戳，所有能够将数据写入OpenTSDB的工具都可以在Akumuli上正常工作。此外，传递ISO 8601-格式的日期时间或纳秒精度时间戳也可以正常工作。

可以在配置中禁用OpenTSDB端点\(在 `akumulid --init` 命令生成的配置中默认启用\)。注意，由于协议冗长，Akumuli中的OpenTSDB端点比原生TCP端点略慢。

### 与OpenTSDB的区别

当使用OpenTSDB时，不得不在数据库模式设计上耗费很长时间。OpenTSDB将所有具有相同度量名的序列存储在一起，所以应该注意不要引入过多具有相同度量名的序列。而对于Akumuli而言，这是不需要的，因为它单独存储每个序列。

Akumuli已经实现 `PUT`命令，其他命令将被忽略，如 `histogram` 或 `rollup`。`version`命令返回一个字符串，指示使用的Akumuli端点。


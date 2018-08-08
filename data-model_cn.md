# 数据模型

## 数据模型

Akumuli的数据模型设计用来跟踪现实世界对象的属性，假设这些属性可以真实化，典型例子如房间里边的传感器测量出伴随时间的温度和湿度，其他例子如虚拟服务器上的CPU利用率和空闲时间。

## 对象跟踪

每个对象可以由标签集唯一定位，例如：酒店房间可以通过建筑物和房间号码来标识\(room=42 building=2\)，因此可以使用两个标签定位特定房间。

标签可以看作对象的地址，但它们并不局限于此。可以通过添加冗余标签来提升搜索能力。在我们前面的例子中，可以添加像楼层号和建筑物侧厅\(room=42 building=2 floor=1 wing=East\)等冗余信息，以便我们可以选择建筑物某些特定部分的所有房间。Akumuli的查询系统允许只使用标签子集来选择对象\(如果这个子集对于每个对象都是惟一的\)，例如，即使有“楼层”和“侧厅”标记，仍然可以使用两个标签"房间"和“建筑物”来指定特定房间，这意味着不必创建唯一ID或GUID来跟踪对象\(如果确实想要，仍然可以将它们添加为标签\)。

## 属性

属性是可测量对象的一个方面，例如，可以测量房间内的温度、电路中的电压以及磁盘上的空闲空间，每个属性都有一个名称通常称为度量名，例如“温度”、“电压”、“磁盘空闲”。度量名应该在对象中是唯一的，假设，如果想使用两个传感器来测量房间里的一氧化碳量，那么应该使用两个不同的测量名称来区分两个传感器，否则，将无法区分两种传感器的测量结果：

```text
colevel_A room=42 building=2 floor=1 wing=East room_type=Luxe
colevel_B room=42 building=2 floor=1 wing=East room_type=Luxe
```

另一种方式是使用标签\(但是在本例中，序列数据将对应于两个不同对象\)：

```text
colevel room=42 building=2 floor=1 wing=East room_type=Luxe sensor=A
colevel room=42 building=2 floor=1 wing=East room_type=Luxe sensor=B
```

## 序列名

度量名和标签集定义了唯一的序列名，Akumuli希望度量名后面有标签集，标签顺序没有特别要求，下边是一个序列名的例子：

```text
temperature room=42 building=2 floor=1 wing=East room_type=Luxe
humidity room=42 building=2 floor=1 wing=East room_type=Luxe
colevel room=42 building=2 floor=1 wing=East room_type=Luxe
```

这是三个序列名，它们都对应于同一对象\(房间\)，但属性不同。第一个跟踪房间温度，第二个跟踪湿度，最后一个是房间的一氧化碳水平。

来自DevOps监视数据序列名的另一个例子：

```text
mem.commit OS=Ubuntu_16.04 region=ap-southeast-1 host=PG-mirror host_IP=172.16.254.1 team=NJ instance-type=m3.2xlarge arch=x64 rack=64
```

它跟踪服务器的内存使用情况。注意，这组标签有很大程度的冗余，只有“host”或者只有“host_IP”标签都可以唯一标识这个序列；而其它所有标签可以用于提供更加充足的分析，例如，可能想知道某种实例类型使用的平均内存。另外也请注意，只有当两个序列名的标签相等时，它们才对应于同一对象。这也意味着，如果将一个标签添加到序列名中，则序列名将变得不同。但是，如果要将多个序列合并为一个序列，可以忽略查询中的一些标记，Akumuli的查询语言允许这样做。因此，如果在房间里有多个一氧化碳传感器：

```text
colevel room=42 building=2 floor=1 wing=East sensor=A
colevel room=42 building=2 floor=1 wing=East sensor=B
```

可以使用查询语言将这两个序列合并为一个包含来自两个传感器读数的序列。

## 数据点

每个数据点应该包含完整的序列名、时间戳以及数值。如上所述，序列名应该包含度量名和标签集。

```text
<metric-name> <tag1>=<tag-value1> <tag2>=<tag-value2>...<tagN>=<tag-valueN>
```

标签顺序无关紧要，“metric tag1=1 tag2=2”和“metric tag2=2 tag1=1”对应于相同序列。度量名、标签名和标签值可以包含除空格字符以外的任何字符。序列名“mem commit OS=Ubuntu 16.04 region=ap-southeast-1 host=PG-mirror host\_IP=172.16.254.1”是无效的，因为度量名\(mem commit\)和标签值“Ubuntu 16.04”包含空格字符。此外，在“=”符号之前或之后的键值对中不应该有任何空格，标签“region = ap-southeast-1”是错误的，它应该是“region=ap-southeast-1”。序列不需要事先创建，如果使用新的序列名写入数据点，Akumuli将自动创建新的序列。

时间戳可以是简单整数或者ISO 8601格式的日期时间\(Akumuli只支持基本形式的组合日期时间形式，如20170405T123000.000001001)。

每个序列的所有数据点都应按时间戳排序，虽然可以用任何顺序写入不同的序列，但每个序列都应该接收随时间戳递增的数据点。如果已经将时间戳设置为20170405T123000.099写入了某些序列的数据点，然后尝试写入一个新的数据点，其时间戳字段设置为20170405T122959.001，会得到“延迟写入”错误。时间戳的值可以是整数或浮点数，建议的格式化方法是使用“%.17g”格式字符串\(使用printf语法\)或其它等效项，保证精度不会丢失。


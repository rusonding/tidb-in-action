## SQL 层简介

TiDB 的 SQL层，即 tidb-server，跟 Google 的 [`F1`](https://dbdb.io/db/google-f1) 层比较类似，负责将 SQL 翻译成 KV 操作，转发给共享的分布式 KV 存储层 TiKV，并组装返回结果，最终返回查询结果。

这一层的节点都是无状态的节点，本身并不存储数据，节点之间完全对等。

### SQL 运算

能想到的最简单的方案就是通过上一节所述的`关系模型 -> Key-Value模型`的映射方案，将 SQL 查询映射为对 KV 的查询，再通过 KV 接口获取对应的数据，最后执行各种计算。

比如`Select count(*) from user where name="TiDB";`这样一个语句，我们需要读取表中所有的数据，然后检查`Name`字段是否是`TiDB`，如果是的话，则返回这一行。具体流程是：

1. 构造出`Key Range`：一个表中所有的`RowID`都在`[0, MaxInt64)`这个范围内，那么我们用`0`和`MaxInt64`根据`Row`的`Key`编码规则，就能构造出一个 `[StartKey, EndKey)`的左闭右开区间。
2. 扫描`Key Range`：根据上面构造出的`Key Range`，读取 TiKV 中的数据
3. 过滤数据：对于读到的每一行数据，计算`name="TiDB"`这个表达式，如果为真，则向上返回这一行，否则丢弃这一行数据
4. 计算`Count`：对符合要求的每一行，累计到`Count`值上面

整个流程示意图如下：

![naive sql flow](http://img.mp.sohu.com/upload/20170524/cbd683354f5a4a03b1ff70e1cee4a520_th.png)


这个方案肯定是可以`Work`的，但是并不能`Work`的很好，原因是显而易见的：

1. 在扫描数据的时候，每一行都要通过 KV 操作同 TiKV 中读取出来，至少有一次 RPC 开销，如果需要扫描的数据很多，那么这个开销会非常大
2. 并不是所有的行都有用，如果不满足条件，其实可以不读取出来
3. 符合要求的行的值并没有什么意义，实际上这里只需要有几行数据这个信息就行

### 分布式 SQL 运算

如何避免上述缺陷也是显而易见的，首先我们需要将计算尽量靠近存储节点，以避免大量的`RPC`调用。其次，我们需要将`Filter`也下推到存储节点进行计算，这样只需要返回有效的行，避免无意义的网络传输。最后，我们可以将聚合函数`Count`也下推到存储节点，进行预聚合，每个节点只需要返回一个`Count`值即可，再由SQL层将`Count`值Sum起来。

这里有一个数据逐层返回的示意图：

![dist sql flow](http://img.mp.sohu.com/upload/20170524/8cbf1c1e550c46688093afcfce6bbdb6_th.png)

### SQL 层架构

通过上面的例子，希望大家对 SQL 语句的处理有一个基本的了解。实际上 TiDB 的 SQL 层要复杂的多，模块以及层次非常多，下面这个图列出了重要的模块以及调用关系：

![tidb sql layer](http://img.mp.sohu.com/upload/20170524/2bc80d3743ad4d029f8a8e6be5a70ec6_th.png)


用户的 SQL 请求会直接或者通过`Load Balancer`发送到 tidb-server，tidb-server 会解析`MySQL Protocol Packet`，获取请求内容，然后做语法解析、查询计划制定和优化、执行查询计划获取和处理数据。数据全部存储在 TiKV 集群中，所以在这个过程中 tidb-server 需要和 TiKV 交互，获取数据。最后 tidb-server 需要将查询结果返回给用户。
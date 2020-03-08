# 3.5 SQL 慢查询系统表

TiDB 会将执行时间超过 slow-threshold（默认值为 300 毫秒）的语句记录到 slow-query-file（默认值："tidb-slow.log"）日志中，用于帮助定位慢查询语句，分析和解决 SQL 执行的性能问题。

TiDB 默认启用慢查询日志，可以修改配置 enable-slow-log 来启用或禁用它。

## 日志示例

```
# Time: 2019-08-14T09:26:59.487776265+08:00
# Txn_start_ts: 410450924122144769
****
# User: root@127.0.0.1
# Conn_ID: 3086
# Query_time: 1.527627037
# Parse_time: 0.000054933
# Compile_time: 0.000129729
# Process_time: 0.07 Request_count: 1 Total_keys: 131073 Process_keys: 131072 Prewrite_time: 0.335415029 Commit_time: 0.032175429 Get_commit_ts_time: 0.000177098 Local_latch_wait_time: 0.106869448 Write_keys: 131072 Write_size: 3538944 Prewrite_region: 1
# DB: test
# Is_internal: false
# Digest: 50a2e32d2abbd6c1764b1b7f2058d428ef2712b029282b776beb9506a365c0f1
# Stats: t:414652072816803841
# Num_cop_tasks: 1
# Cop_proc_avg: 0.07 Cop_proc_p90: 0.07 Cop_proc_max: 0.07 Cop_proc_addr: 172.16.5.87:20171
# Cop_wait_avg: 0 Cop_wait_p90: 0 Cop_wait_max: 0 Cop_wait_addr: 172.16.5.87:20171
# Mem_max: 525211
# Succ: true
# Plan: tidb_decode_plan('ZJAwCTMyXzcJMAkyMAlkYXRhOlRhYmxlU2Nhbl82CjEJMTBfNgkxAR0AdAEY1Dp0LCByYW5nZTpbLWluZiwraW5mXSwga2VlcCBvcmRlcjpmYWxzZSwgc3RhdHM6cHNldWRvCg==')
insert into t select * from t;
```

## 字段含义说明

> 注意：
> 慢查询日志中所有时间相关字段的单位都是 “秒”

Slow Query 基础信息：

* Time：表示日志打印时间。
* Query_time：表示执行这个语句花费的时间。
* Parse_time：表示这个语句在语法解析阶段花费的时间。
* Compile_time：表示这个语句在查询优化阶段花费的时间。
* Query：表示 SQL 语句。慢日志里面不会打印 Query，但映射到内存表后，对应的字段叫 Query。
* Digest：表示 SQL 语句的指纹。
* Stats：表示 table 使用的统计信息版本时间戳。如果时间戳显示为 pseudo，表示用默认假设的统计信息。
* Txn_start_ts：表示事务的开始时间戳，也是事务的唯一 ID，可以用这个值在 TiDB 日志中查找事务相关的其他日志。
* Is_internal：表示是否为 TiDB 内部的 SQL 语句。true 表示 TiDB 系统内部执行的 SQL 语句，false 表示用户执行的 SQL 语句。
* Index_ids：表示语句涉及到的索引的 ID。
* Succ：表示语句是否执行成功。
* Backoff_time：表示语句遇到需要重试的错误时在重试前等待的时间，常见的需要重试的错误有以下几种：遇到了 lock、Region 分裂、tikv server is busy。
* Plan：表示语句的执行计划，用 select tidb_decode_plan('xxx...') SQL 语句可以解析出具体的执行计划。

和事务执行相关的字段：

* Prewrite_time：表示事务两阶段提交中第一阶段（prewrite 阶段）的耗时。
* Commit_time：表示事务两阶段提交中第二阶段（commit 阶段）的耗时。
* Get_commit_ts_time：表示事务两阶段提交中第二阶段（commit 阶段）获取 commit 时间戳的耗时。
* Local_latch_wait_time：表示事务两阶段提交中第二阶段（commit 阶段）发起前在 TiDB 侧等锁的耗时。
* Write_keys：表示该事务向 TiKV 的 Write CF 写入 Key 的数量。
* Write_size：表示事务提交时写 key 或 value 的总大小。
* Prewrite_region：表示事务两阶段提交中第一阶段（prewrite 阶段）涉及的 TiKV Region 数量。每个 Region 会触发一次远程过程调用。

和内存使用相关的字段：

* Memory_max：表示执行期间 TiDB 使用的最大内存空间，单位为 byte。

和 SQL 执行的用户相关的字段：

* User：表示执行语句的用户名。
* Conn_ID：表示用户的链接 ID，可以用类似 con:3 的关键字在 TiDB 日志中查找该链接相关的其他日志。
* DB：表示执行语句时使用的 database。

和 TiKV Coprocessor Task 相关的字段：

* Request_count：表示这个语句发送的 Coprocessor 请求的数量。
* Total_keys：表示 Coprocessor 扫过的 key 的数量。
* Process_time：执行 SQL 在 TiKV 的处理时间之和，因为数据会并行的发到 TiKV 执行，这个值可能会超过 Query_time。
* Wait_time：表示这个语句在 TiKV 的等待时间之和，因为 TiKV 的 Coprocessor 线程数是有限的，当所有的 Coprocessor 线程都在工作的时候，请求会排队；当队列中有某些请求耗时很长的时候，后面的请求的等待时间都会增加。
* Process_keys：表示 Coprocessor 处理的 key 的数量。相比 total_keys，processed_keys 不包含 MVCC 的旧版本。如果 processed_keys 和 total_keys 相差很大，说明旧版本比较多。
* Cop_proc_avg：cop-task 的平均执行时间。
* Cop_proc_p90：cop-task 的 P90 分位执行时间。
* Cop_proc_max：cop-task 的最大执行时间。
* Cop_proc_addr：执行时间最长的 cop-task 所在地址。
* Cop_wait_avg：cop-task 的平均等待时间。
* Cop_wait_p90：cop-task 的 P90 分位等待时间。
* Cop_wait_max：cop-task 的最大等待时间。
* Cop_wait_addr：等待时间最长的 cop-task 所在地址。

## 慢查询系统表
通过查询 INFORMATION_SCHEMA.SLOW_QUERY 表来查询当前节点的慢查询日志中的内容，表中列名和慢日志中字段名一一对应，表结构可查看 Information Schema 中关于 SLOW_QUERY 表的介绍。

TiDB 4.0 中的 SLOW_QUERY 已经支持查询任意时间段的慢日志，即支持查询已经被切换的慢日志文件的数据。用户查询时只需要指定 TIME 时间范围即可定位需要解析的慢日志文件。如果查询不指定时间范围，则和 4.0 版本之前行为一致，只解析当前的慢日志文件。

TiDB 4.0 中新增了 CLUSTER_SLOW_QUERY 系统表，用来查询所有 TiDB 节点的 SLOW_QUERY 数据，使用上和 SLOW_QUERY 是一样的。

> 注意：
> 每次查询 SLOW_QUERY 表时，TiDB 都会去读取和解析一次当前节点的慢查询日志。

## 查询 SLOW_QUERY 示例

### 搜索 Top N 的慢查询

查询 Top 2 的慢查询。is_internal=false 表示排除 TiDB 内部的慢查询：
```
select query_time, query
from information_schema.slow_query
where is_internal = false  -- 排除 TiDB 内部的慢查询 SQL
order by query_time desc
limit 2;
```

输出样例：

```
+--------------+------------------------------------------------------------------+
| query_time   | query                                                            |
+--------------+------------------------------------------------------------------+
| 12.77583857  | select * from t_slim, t_wide where t_slim.c0=t_wide.c0;          |
|  0.734982725 | select t0.c0, t1.c1 from t_slim t0, t_wide t1 where t0.c0=t1.c0; |
+--------------+------------------------------------------------------------------+
```

### 搜索某个用户的 Top N 慢查询

下面例子中搜索 test 用户执行的慢查询 SQL，且按执行消耗时间逆序排序显式前 2 条：

```
select query_time, query, user
from information_schema.slow_query
where is_internal = false  -- 排除 TiDB 内部的慢查询 SQL
  and user = "test"        -- 查找的用户名
order by query_time desc
limit 2;
```

输出样例：

```
+-------------+------------------------------------------------------------------+----------------+
| Query_time  | query                                                            | user           |
+-------------+------------------------------------------------------------------+----------------+
| 0.676408014 | select t0.c0, t1.c1 from t_slim t0, t_wide t1 where t0.c0=t1.c1; | test           |
+-------------+------------------------------------------------------------------+----------------+
```

### 根据 SQL 指纹搜索同类慢查询

在得到 Top N 的慢查询 SQL 后，可通过 SQL 指纹继续搜索同类慢查询 SQL。
先获取 Top N 的慢查询和对应的 SQL 指纹：

```
select query_time, query, digest
from information_schema.slow_query
where is_internal = false
order by query_time desc
limit 1;
```

输出样例：

```
+-------------+-----------------------------+------------------------------------------------------------------+
| query_time  | query                       | digest                                                           |
+-------------+-----------------------------+------------------------------------------------------------------+
| 0.302558006 | select * from t1 where a=1; | 4751cb6008fda383e22dacb601fde85425dc8f8cf669338d55d944bafb46a6fa |
+-------------+-----------------------------+------------------------------------------------------------------+
```

再根据 SQL 指纹搜索同类慢查询：

```
select query, query_time
from information_schema.slow_query
where digest = "4751cb6008fda383e22dacb601fde85425dc8f8cf669338d55d944bafb46a6fa";
```

输出样例：

```
+-----------------------------+-------------+
| query                       | query_time  |
+-----------------------------+-------------+
| select * from t1 where a=1; | 0.302558006 |
| select * from t1 where a=2; | 0.401313532 |
+-----------------------------+-------------+
```

### 搜索统计信息为 pseudo 的慢查询 SQL 语句

```
select query, query_time, stats
from information_schema.slow_query
where is_internal = false
  and stats like '%pseudo%';
```

输出样例：

```
+-----------------------------+-------------+---------------------------------+
| query                       | query_time  | stats                           |
+-----------------------------+-------------+---------------------------------+
| select * from t1 where a=1; | 0.302558006 | t1:pseudo                       |
| select * from t1 where a=2; | 0.401313532 | t1:pseudo                       |
| select * from t1 where a>2; | 0.602011247 | t1:pseudo                       |
| select * from t1 where a>3; | 0.50077719  | t1:pseudo                       |
| select * from t1 join t2;   | 0.931260518 | t1:407872303825682445,t2:pseudo |
+-----------------------------+-------------+---------------------------------+
```

### 查询集群各个 TIDB 节点的慢查询数量

```
select instance, count(*) from information_schema.cluster_slow_query where time >= "2020-03-06 00:00:00" and time < now() group by instance;
```

输出样例：

```
+---------------+----------+
| instance      | count(*) |
+---------------+----------+
| 0.0.0.0:10081 | 124      |
| 0.0.0.0:10080 | 119771   |
+---------------+----------+
```

## 解析其他的 TiDB 慢日志文件

在 TiDB 4.0 之前，由于只支持解析 当前的慢日志文件，如果需要解析其他的慢日志文件，可以通过设置 session 变量 tidb_slow_query_file 控制查询 INFORMATION_SCHEMA.SLOW_QUERY 时要读取和解析的文件，示例如下：

```
set tidb_slow_query_file = "/path-to-log/tidb-slow.log"
```

由于 TiDB 4.0 已经支持解析任意时间段的慢日志，所以几乎不需要上面的 session 变量了。

## 用 pt-query-digest 工具分析 TiDB 慢日志

可以用 pt-query-digest 工具分析 TiDB 慢日志。

> **注意：**
> 建议使用 pt-query-digest 3.0.13 及以上版本。

示例如下：

```
pt-query-digest --report tidb-slow.log
```

输出样例：

```
# 320ms user time, 20ms system time, 27.00M rss, 221.32M vsz
# Current date: Mon Mar 18 13:18:51 2019
# Hostname: localhost.localdomain
# Files: tidb-slow.log
# Overall: 1.02k total, 21 unique, 0 QPS, 0x concurrency _________________
# Time range: 2019-03-18-12:22:16 to 2019-03-18-13:08:52
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time           218s    10ms     13s   213ms    30ms      1s    19ms
# Query size       175.37k       9   2.01k  175.89  158.58  122.36  158.58
# Commit time         46ms     2ms     7ms     3ms     7ms     1ms     3ms
# Conn ID               71       1      16    8.88   15.25    4.06    9.83
# Process keys     581.87k       2 103.15k  596.43  400.73   3.91k  400.73
# Process time         31s     1ms     10s    32ms    19ms   334ms    16ms
# Request coun       1.97k       1      10    2.02    1.96    0.33    1.96
# Total keys       636.43k       2 103.16k  652.35  793.42   3.97k  400.73
# Txn start ts     374.38E       0  16.00E 375.48P   1.25P  89.05T   1.25P
# Wait time          943ms     1ms    19ms     1ms     2ms     1ms   972us
.
.
.
```

### 定位问题语句的方法

SLOW_QUERY 中的语句并不是都是有问题的。造成集群整体压力增大的是那些 process_time 很大的语句。如果wait_time 很大，但 process_time 很小的语句通常不是问题语句，而是因为被问题语句阻塞，在执行队列等待造成的响应时间过长。

## admin show slow 命令

除了基于TiDB 日志，还有一种定位慢查询的方式是通过 admin show slow SQL 命令：

> 注意:
> 此命令仅显示当前TiDB节点的慢查询

```
admin show slow recent N;
admin show slow top [internal | all] N;
```

recent N 会显示最近的 N 条慢查询记录，例如：

```
admin show slow recent 10;
```

top N 则显示最近一段时间（大约几天）内，最慢的查询记录。如果指定 internal 选项，则返回查询系统内部 SQL 的慢查询记录；如果指定 all 选项，返回包含系统内部的所有SQL 汇总以后的慢查询记录；默认只返回非系统内部的SQL中的慢查询记录。

>说明：
>N的最大值是30，显示时间范围为最近7天

例如显示最慢的3条SQL：

```
admin show slow top 3;
```

输出样例：

```
+---------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+----------+--------------------------------------+------+---------+--------------------+----------------+--------------------+-----------------------+-----------+----------+------------------------------------------------------------------+
| SQL                                                                                                                                         | START                      | DURATION | DETAILS                              | SUCC | CONN_ID | TRANSACTION_TS     | USER           | DB                 | TABLE_IDS             | INDEX_IDS | INTERNAL | DIGEST                                                           |
+---------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+----------+--------------------------------------+------+---------+--------------------+----------------+--------------------+-----------------------+-----------+----------+------------------------------------------------------------------+
| select instance, count(*) from `CLUSTER_SLOW_QUERY` where time >= "2020-03-05 20:55:00" and time < "2020-03-05 20:57:00" group by instance  | 2020-03-07 18:14:48.815964 | 0:00:03  | Backoff_time: 0.077 Request_count: 2 | 1    | 4       | 415124970215833601 | root@127.0.0.1 | information_schema | [4611686018427387951] |           | 0        | 9b4f3ab5d876d60b89d74a0023850a09f35689014a29ef2b5a83f79cfaba8137 |
| select instance, count(*) from information_schema.cluster_slow_query where time >= "2020-03-06 00:00:00" and time < now() group by instance | 2020-03-07 18:21:27.653822 | 0:00:02  | Request_count: 2                     | 1    | 4       | 415125074771968002 | root@127.0.0.1 | information_schema | [4611686018427387951] |           | 0        | de3ef2894becb6f562cfaf6234339c86573688637686b45c4a0262be3b8095c8 |
| select instance, count(*) from `CLUSTER_SLOW_QUERY` where time >= "2020-03-06 00:00:00" and time < now() group by instance                  | 2020-03-07 18:19:28.689143 | 0:00:02  | Request_count: 2                     | 1    | 4       | 415125043590987777 | root@127.0.0.1 | information_schema | [4611686018427387951] |           | 0        | 5cfb4b56d41e12ce3674ef069c568deb7dbedf14a2d5745055f66f63d40c72cb |
+---------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+----------+--------------------------------------+------+---------+--------------------+----------------+--------------------+-----------------------+-----------+----------+------------------------------------------------------------------+
```

由于内存限制，保留的慢查询记录的条数是有限的。当命令查询的 N 大于记录条数时，返回的结果记录条数会小于 N。

输出内容详细说明，如下：

| 列名           | 描述                                   |
| :------------- | :------------------------------------- |
| start          | SQL 语句执行开始时间                   |
| duration       | SQL 语句执行持续时间                   |
| details        | 执行语句的详细信息                     |
| succ           | SQL 语句执行是否成功，1: 成功，0: 失败 |
| conn_id        | session 连接 ID                        |
| transcation_ts | 事务提交的 commit ts                   |
| user           | 执行该语句的用户名                     |
| db             | 执行该 SQL 涉及到 database             |
| table_ids      | 执行该 SQL 涉及到表的 ID               |
| index_ids      | 执行该 SQL 涉及到索引 ID               |
| internal       | 表示为 TiDB 内部的 SQL 语句            |
| digest         | 表示 SQL 语句的指纹                    |
| sql            | 执行的 SQL 语句                        |

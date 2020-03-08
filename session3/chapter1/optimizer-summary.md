# 优化器模型

优化器的作用是在合理的时间内找到合理的执行计划。TiDB 采用了 System R 的优化器模型，优化过程分为逻辑优化和物理优化两个阶段。逻辑优化过程顺序的遍历内部实现定义好的优化规则，不断的改写 SQL 的逻辑执行计划。物理优化则是将改写后逻辑执行计划变成可以执行的物理执行计划，这一过程会决定用什么索引读表，用什么样的顺序进行多表 Join，用什么算法做每一个 Join 操作等。

## 常见逻辑优化规则简介

### 基本逻辑算子介绍

TiDB 中的逻辑算子主要以下几个：
* DataSource：数据源，代表一个源表，`select * from t` 里面的 `t`。
* Selection： 代表了相应的过滤条件，`select * from t where a = 5` 里面的 `where a = 5`。
* Projection：投影操作，也用于表达式计算， `select c, a +b from t` 里面的 `c` 和 `a + b`就是投影和表达式计算操作。
* Join：两个表的连接操作，`select t1.b, t2.c from t1 join t2 on t1.a = t2.a` 中的 `t1 join t2 on t1.a = t2.a` 就是两个表 `t1` 和 `t2` 的连接操作。Join 有内连接，左连接，右连接等多种连接方式。

Selection，Projection，Join（简称 SPJ） 是最基本的 3 种算子。

### 常见逻辑优化规则

逻辑优化是基于规则的优化，对输入的逻辑执行计划按顺序应用这些优化规则，从而使整个逻辑执行计划变得更加高效。这些常用逻辑优化规则包括：


| 列表 1 |  列表 2 |
|-------|---------|
| 1、列裁剪	| 6、外连接转内连接|
| 2、分区剪裁 |	7、子查询去关联|
| 3、聚合消除 |	8、谓词下推 |
| 4、Max / Min优化	| 9、聚合下推 |
| 5、外连接消除	| 10、TopN / Limit 下推|

逻辑优化部分示例如下：

#### 例子1：外连接消除

外连接消除指的是将整个连接操作从查询中移除。外连接消除需要满足一定条件：
* 条件 1：LogicalJoin 的父亲算子只会用到 LogicalJoin 的 outer plan 所输出的列
* 条件 2：
  * 条件 2.1：LogicalJoin 中的 join key 在 inner plan 的输出结果中满足唯一性属性
  * 条件 2.2：LogicalJoin 的父亲算子会对输入的记录去重

条件 1 和条件 2 必须同时满足，但条件 2.1 和条件 2.2 只需满足一条即可。

满足条件 1 和 条件 2.1 的一个例子：

```sql
select t1.a from t1 left join t2 on t1.b = t2.b;
```

可以被改写成：

```sql
select t1.a from t1;
```

#### 例子3：`Max`/`Min` 优化

`Max`/`Min` 优化，会对 `Max`/`Min` 语句进行改写。如下面的语句：

```sql
select min(id) from t;
```

改成下面的写法，可以实现类似的效果：

```sql
select id from t order by id desc limit 1;
```

前一个语句生成的执行计划，是一个 TableScan 上面接一个 Aggregation，这是一个全表扫描的操作。后一个语句，生成执行计划是 TableScan + Sort + Limit。通常数据表中的 id 列是主键或者是存在索引，数据本身有序，这样 Sort 就可以消除，最终变成 TableScan/IndexLookUp + Limit，这样就避免了全表扫描的操作，只需要读到第一条数据就能返回结果。

最大最小消除由优化器“自动”地做这个变换。

## 物理优化原理

物理优化是基于代价的优化，为逻辑优化阶段产生的逻辑执行计划制定物理执行计划。这一阶段中，优化器会为逻辑执行计划中的每个算子选择具体的物理实现。逻辑算子的不同物理实现有着不同的时间复杂度、资源消耗和物理属性等。在这个过程中，优化器会根据数据的统计信息来确定不同物理实现的代价，并选择整体代价最小的物理执行计划。

物理优化需要做的决策：
* 选择索引还是全表扫
* 选择哪个索引
* 逻辑算子的物理实现
* 是否将算子下推到存储层

## 统计信息的收集与维护

TiDB 优化器会根据统计信息来选择最优的执行计划。统计信息收集了表级别和列级别的信息，表的统计信息包括总行数和修改的行数。列的统计信息包括不同值的数量、NULL 的数量、直方图、列上出现次数最多的值 TOPN 以及该列的 Count-Min Sketch 信息。

### 手动搜集

通过执行 `ANALYZE` 语句来收集统计信息。如需更快的分析速度，可将 `tidb_enable_fast_analyze`（默认值为 0）设置为 1 来打开快速分析功能。以数据库中 person 表为例，使用 fast analyze 的执行语句如下：

```sql
set @@tidb_enable_fast_analyze = 1;
analyze table person;
```

收集统计信息过程中，可以通过 `show analyze status` 语句查询执行状态，该语句也可以通过 `where` 子句对输出结果进行筛选，显示输出结果如下：

```sql
mysql> show analyze status where job_info = 'analyze columns';
+--------------+------------+-----------------+---------------------+----------+
| Table_schema | Table_name | Job_info        | Start_time          | State    |
+--------------+------------+-----------------+---------------------+----------+
| test         | person     | analyze columns | 2020-03-07 06:22:34 | finished |
| test         | customer   | analyze columns | 2020-03-07 06:32:19 | finished |
| test         | person     | analyze columns | 2020-03-07 06:35:27 | finished |
+--------------+------------+-----------------+---------------------+----------+
3 rows in set (0.01 sec)
```

### 自动更新

在执行 DML 语句时，TiDB 会自动更新表的总行数以及修改的行数。这些信息会定期自动持久化，更新周期是 1 分钟（20 * stats-lease）

> 注意：stats-lease 的默认值是 3s，如果将其设定为 0，则关闭统计信息自动更新。

### 统计信息查看

查看表的统计信息 meta 信息：

```sql
mysql> show stats_meta where table_name = 'person';
+---------+------------+----------------+---------------------+--------------+-----------+
| Db_name | Table_name | Partition_name | Update_time         | Modify_count | Row_count |
+---------+------------+----------------+---------------------+--------------+-----------+
| test    | person     |                | 2020-03-07 07:20:54 |            0 |         4 |
+---------+------------+----------------+---------------------+--------------+-----------+
1 row in set (0.01 sec)
```

查看表的健康度信息：

```sql
mysql> show stats_healthy where table_name = 'person';
+---------+------------+----------------+---------+
| Db_name | Table_name | Partition_name | Healthy |
+---------+------------+----------------+---------+
| test    | person     |                |     100 |
+---------+------------+----------------+---------+
1 row in set (0.00 sec)
```

可通过 `SHOW STATS_HISTOGRAMS` 来查看列的不同值数量以及 NULL 数量等信息:

```sql
mysql> show stats_histograms where table_name = 'person';
+---------+------------+----------------+-------------+----------+---------------------+----------------+------------+--------------+-------------+
| Db_name | Table_name | Partition_name | Column_name | Is_index | Update_time         | Distinct_count | Null_count | Avg_col_size | Correlation |
+---------+------------+----------------+-------------+----------+---------------------+----------------+------------+--------------+-------------+
| test    | person     |                | name        |        0 | 2020-03-07 07:20:54 |              4 |          0 |         6.25 |        -0.2 |
+---------+------------+----------------+-------------+----------+---------------------+----------------+------------+--------------+-------------+
1 row in set (0.00 sec)
```

可通过 `SHOW STATS_BUCKETS` 来查看直方图每个桶的信息：

```sql
mysql> show stats_buckets;
+---------+------------+----------------+-------------+----------+-----------+-------+---------+-------------+-------------+
| Db_name | Table_name | Partition_name | Column_name | Is_index | Bucket_id | Count | Repeats | Lower_Bound | Upper_Bound |
+---------+------------+----------------+-------------+----------+-----------+-------+---------+-------------+-------------+
| test    | person     |                | name        |        0 |         0 |     1 |       1 | jack        | jack        |
| test    | person     |                | name        |        0 |         1 |     2 |       1 | peter       | peter       |
| test    | person     |                | name        |        0 |         2 |     3 |       1 | smith       | smith       |
| test    | person     |                | name        |        0 |         3 |     4 |       1 | tom         | tom         |
+---------+------------+----------------+-------------+----------+-----------+-------+---------+-------------+-------------+
4 rows in set (0.01 sec)
```

### 删除统计信息

可通过执行 `DROP STATS` 语句来删除统计信息。语句如下：

```sql
mysql> DROP STATS person;
```

### 统计信息导入导出

#### 统计信息导出

通过以下接口可以获取数据库 `${db_name}` 中的表 `${table_name}` 的 json 格式的统计信息：

```
http://${tidb-server-ip}:${tidb-server-status-port}/stats/dump/${db_name}/${table_name}
```

示例：获取本机上 test 数据库中 person 表的统计信息：

```sh
curl -G "http://127.0.0.1:10080/stats/dump/test/person" > person.json
```

#### 统计信息导入

将统计信息导出接口得到的 json 文件导入数据库中：

```sql
mysql> LOAD STATS 'file_name';
```

file_name 为导入的统计信息文件名。

# 执行计划概览

阅读和理解执行计划对于 `SQL` 调优非常重要。本小结将向大家介绍如何查看 `SQL` 执行计划。

## 使用 EXPLAIN 语句查看执行计划

执行计划由一系列的算子构成。和其他数据库一样，在 TiDB 中可通过 `EXPLAIN` 语句返回的结果查看某条 `SQL` 的执行计划。

目前 `TiDB` 的 `EXPLAIN` 会输出 4 列，分别是：`id`，`estRows`，`task`，`operator info`。执行计划中每个算子都由这 4 列属性来描述，`EXPLAIN`结果中每一行描述一个算子。每个属性的具体含义如下：

| 属性名          | 含义 |
|:----------------|:----------------------------------------------------------------------------------------------------------|
| id            | 算子的 ID，在整个执行计划中唯一的标识一个算子。在 TiDB 2.1 中，ID 会格式化的显示算子的树状结构。数据从孩子结点流向父亲结点，每个算子的父亲结点有且仅有一个。                                                                                       |
| estRows       | 预计当前算子将会输出的数据条数，基于统计信息以及算子的执行逻辑估算而来。在 4.0 之前叫 count。 |
| task          | 当前这个算子属于什么 task。目前的执行计划分成为两种 task，一种叫 **root** task，在 tidb-server 上执行，一种叫 **cop** task，并行的在 TiKV 或者 TiFlash 上执行。当前的执行计划在 task 级别的拓扑关系是一个 root task 后面可以跟许多 cop task，root task 使用 cop task 的输出结果作为输入。cop task 中执行的也即是 TiDB 下推到 TiKV 或者 TiFlash 上的任务，每个 cop task 分散在 TiKV 或者 TiFlash 集群中，由多个进程共同执行。 |
| operator info | 每个算子的详细信息。各个算子的 operator info 各有不同，可参考下面的示例解读。                   |

## EXPLAIN ANALYZE 输出格式

和 `EXPLAIN` 不同，`EXPLAIN ANALYZE` 会执行对应的 `SQL` 语句，记录其运行时信息，和执行计划一并返回出来，可以视为 `EXPLAIN` 语句的扩展。`EXPLAIN ANALYZE` 语句的返回结果中增加了 `actRows`, `execution info`,`memory`,`disk` 这几列信息：

| 属性名          | 含义 |
|:----------------|:---------------------------------|
| estRows       | 当前算子实际输出的数据条数。 |
| execution info  | time 显示从进入算子到离开算子的全部 wall time，包括所有子算子操作的全部执行时间。如果该算子被父算子多次调用 (loops)，这个时间就是累积的时间。loops 是当前算子被父算子调用的次数。 rows 是当前算子返回的行的总数。|
| memory  | 当前算子占用内存大小 |
| disk  | 当前算子占用磁盘大小 |

一个例子：

```
mysql> explain analyze select * from t where a < 10;
+-------------------------------+---------+---------+-----------+-----------------------------------------------------+--------------------------------------------------------------------------------+---------------+------+
| id                            | estRows | actRows | task      | operator info                                       | execution info                                                                 | memory        | disk |
+-------------------------------+---------+---------+-----------+-----------------------------------------------------+--------------------------------------------------------------------------------+---------------+------+
| IndexLookUp_10                | 9.00    | 9       | root      |                                                     | time:641.245µs, loops:2, rows:9, rpc num: 1, rpc time:242.648µs, proc keys:0   | 9.23046875 KB | N/A  |
| ├─IndexRangeScan_8(Build)     | 9.00    | 9       | cop[tikv] | table:t, index:a, range:[-inf,10), keep order:false | time:142.94µs, loops:10, rows:9                                                | N/A           | N/A  |
| └─TableRowIDScan_9(Probe)     | 9.00    | 9       | cop[tikv] | table:t, keep order:false                           | time:141.128µs, loops:10, rows:9                                               | N/A           | N/A  |
+-------------------------------+---------+---------+-----------+-----------------------------------------------------+--------------------------------------------------------------------------------+---------------+------+
3 rows in set (0.00 sec)
```

从上述例子中可以看出，优化器估算的 `estRows` 和实际执行中统计得到的 `actRows` 几乎是相等的，说明优化器估算误差很小。同时 `IndexLookUp_10` 算子在实际执行过程中使用了约 `9 KB` 的内存，该 `SQL` 在执行过程中，没有触发过任何算子的落盘操作。

## 如何阅读算子的执行顺序

TiDB 的执行计划是一个树形结构，树中每个节点即是算子。考虑到每个算子内多线程并发执行的情况，在一条 `SQL` 执行的过程中，如果能够有一个手术刀把这棵树切开看看，大家可能会发现所有的算子都正在消耗 `CPU` 和内存处理数据，从这个角度来看，算子是没有执行顺序的。

但是如果从一行数据先后被哪些算子处理的角度来看，一条数据在算子上的执行是有顺序的。这个顺序可以通过下面这个规则简单总结出来：

**`Build`总是先于 `Probe` 执行，并且 `Build` 总是出现 `Probe` 前面**

这个原则的前半句是说：如果一个算子有多个孩子结点，孩子结点 ID 后面有 `Build` 关键字的算子总是先于有 `Probe` 关键字的算子执行。后半句是说：TiDB 在展现执行计划的时候，`Build` 端总是第一个出现，接着才是 `Probe` 端。

一些例子：

```
TiDB(root@127.0.0.1:test) > explain select * from t use index(idx_a) where a = 1;
+-------------------------------+---------+-----------+---------------------------------------------------------------+
| id                            | estRows | task      | operator info                                                 |
+-------------------------------+---------+-----------+---------------------------------------------------------------+
| IndexLookUp_7                 | 10.00   | root      |                                                               |
| ├─IndexRangeScan_5(Build)     | 10.00   | cop[tikv] | table:t, index:a, range:[1,1], keep order:false, stats:pseudo |
| └─TableRowIDScan_6(Probe)     | 10.00   | cop[tikv] | table:t, keep order:false, stats:pseudo                       |
+-------------------------------+---------+-----------+---------------------------------------------------------------+
3 rows in set (0.00 sec)
```

这里 `IndexLookUp_7` 算子有两个孩子结点：`IndexRangeScan_5(Build)` 和 `TableRowIDScan_6(Probe)`。可以看到，`IndexRangeScan_5(Build)` 是第一个出现的，并且基于上面这条规则，要得到一条数据，需要先执行它得到一个 `RowID` 以后再由 `TableRowIDScan_6(Probe)` 根据前者读上来的 `RowID` 去获取完整的一行数据。

这种规则隐含的另一个信息是：出现在最前面的算子可能是最先被执行的，而出现在最末尾的算子可能是最后被执行的。比如下面这个例子：

```
TiDB(root@127.0.0.1:test) > explain select * from t t1 use index(idx_a) join t t2 use index() where t1.a = t2.a;
+----------------------------------+----------+-----------+------------------------------------------------------------------+
| id                               | estRows  | task      | operator info                                                    |
+----------------------------------+----------+-----------+------------------------------------------------------------------+
| HashLeftJoin_22                  | 12487.50 | root      | inner join, inner:TableReader_26, equal:[eq(test.t.a, test.t.a)] |
| ├─TableReader_26(Build)          | 9990.00  | root      | data:Selection_25                                                |
| │ └─Selection_25                 | 9990.00  | cop[tikv] | not(isnull(test.t.a))                                            |
| │   └─TableFullScan_24           | 10000.00 | cop[tikv] | table:t2, keep order:false, stats:pseudo                         |
| └─IndexLookUp_29(Probe)          | 9990.00  | root      |                                                                  |
|   ├─IndexFullScan_27(Build)      | 9990.00  | cop[tikv] | table:t1, index:a, keep order:false, stats:pseudo                |
|   └─TableRowIDScan_28(Probe)     | 9990.00  | cop[tikv] | table:t1, keep order:false, stats:pseudo                         |
+----------------------------------+----------+-----------+------------------------------------------------------------------+
7 rows in set (0.00 sec)
```

要完成 `HashLeftJoin_22`，需要先执行 `TableReader_26(Build)` 再执行 `IndexLookUp_29(Probe)`。而在执行 `IndexLookUp_29(Probe)` 的时候，又需要先执行 `IndexFullScan_27(Build)` 再执行 `TableRowIDScan_28(Probe)`。所以从整条执行链路来看，`TableRowIDScan_28(Probe)` 是最后被唤起执行的。

## 如何阅读扫表的执行计划

真正执行扫表（读盘或者读 TiKV Block Cache）操作的算子有如下几类：

- **TableFullScan**：这是大家所熟知的 “全表扫” 操作
- **TableRangeScan**：带有范围的表数据扫描操作，通常扫描的数据量不大
- **TableRowIDScan**：根据上层传递下来的 `RowID` 精确的扫描表数据的算子
- **IndexFullScan**：另一种 “全表扫”，只不过这里扫的是索引数据，不是表数据
- **IndexRangeScan**：带有范围的索引数据扫描操作，通常扫描的数据量不大

TiDB 会汇聚 TiKV/TiFlash 上扫描的数据或者计算结果，这种 “数据汇聚” 算子目前有如下几类：

- **TableReader**：汇总 TiKV 上底层扫表算子是 `TableFullScan` 或 `TableRangeScan` 的算子。
- **IndexReader**：汇总 TiKV 上底层扫表算子是 `IndexFullScan` 或 `IndexRangeScan` 的算子。
- **IndexLookUp**：先汇总 Build 端 TiKV 扫描上来的 RowID，再去 Probe 端上根据这些 RowID 精确的读取 TiKV 上的数据。Build 端是 `IndexFullScan` 或 `IndexRangeScan`，Probe 端是 `TableRowIDScan`。
- **IndexMerge**：和 IndexLookupReader 类似，可以看做是它的扩展，可以同时读取多个索引的数据，有多个 Build 端，一个 Probe 端。执行过程也很类似，先汇总所有 Build 端 TiKV 扫描上来的 RowID，再去 Probe 端上根据这些 RowID 精确的读取 TiKV 上的数据。Build 端是 `IndexFullScan` 或 `IndexRangeScan`，Probe 端是 `TableRowIDScan`。

**IndexLookUp 示例：**

```
mysql> explain select * from t use index(idx_a);
+-------------------------------+----------+-----------+--------------------------------------------------+
| id                            | estRows  | task      | operator info                                    |
+-------------------------------+----------+-----------+--------------------------------------------------+
| IndexLookUp_6                 | 10000.00 | root      |                                                  |
| ├─IndexFullScan_4(Build)      | 10000.00 | cop[tikv] | table:t, index:a, keep order:false, stats:pseudo |
| └─TableRowIDScan_5(Probe)     | 10000.00 | cop[tikv] | table:t, keep order:false, stats:pseudo          |
+-------------------------------+----------+-----------+--------------------------------------------------+
3 rows in set (0.00 sec)
```

这里 `IndexLookUp_6` 算子有两个孩子结点：`IndexFullScan_4(Build)` 和 `TableRowIDScan_5(Probe)`。可以看到，`IndexFullScan_4(Build)` 执行索引全表扫，扫描索引 `a` 的所有数据，因为是全范围扫，这个操作将获得表中所有数据的 RowID，之后再由 `TableRowIDScan_5(Probe)` 去根据这些 RowID 去扫描所有的表数据。可以预见的是，这个执行计划不如直接使用 TableReader 进行全表扫，因为同样都是全表扫，这里的 IndexLookUp 多扫了一次索引，带来了额外的开销。

**TableReader 示例：**

```
mysql> explain select * from t where a > 1 or b >100;
+-------------------------+----------+-----------+-----------------------------------------+
| id                      | estRows  | task      | operator info                           |
+-------------------------+----------+-----------+-----------------------------------------+
| TableReader_7           | 8000.00  | root      | data:Selection_6                        |
| └─Selection_6           | 8000.00  | cop[tikv] | or(gt(test.t.a, 1), gt(test.t.b, 100))  |
|   └─TableFullScan_5     | 10000.00 | cop[tikv] | table:t, keep order:false, stats:pseudo |
+-------------------------+----------+-----------+-----------------------------------------+
3 rows in set (0.00 sec)
```

在上面例子中 `TableReader_7` 算子的孩子结点是 `Selection_6`。以这个孩子结点为根的子树被当做了一个 Cop Task 下发给了相应的 TiKV，这个 Cop Task 上执行扫表操作的是 `TableFullScan_5`。`Selection` 表示 SQL 语句中的选择条件，可能来自 SQL 语句中的 `WHERE`/`HAVING`/`ON` 子句。由 `TableFullScan_5` 可以看到，这个执行计划使用了一个全表扫的操作，集群的负载将因此而上升，可能会影响到集群中正在运行的其他查询。这时候如果能够建立合适的索引，并且使用 `IndexMerge` 算子，将能够极大的提升查询的性能，降低集群的负载。

**IndexMerge  示例：**

> **注意：**
>
> 目前 TIDB 的 `Index Merge` 特性在 4.0 RC 版本中默认关闭，同时 4.0 中的 `Index Merge` 目前支持的场景仅限于析取范式（`or` 连接的表达式），对合取范式（`and` 连接的表达式）将在之后的版本中支持。
> 开启 `Index Merge` 特性，可通过在客户端中设置 session 或者 global 变量完成：`set @@tidb_enable_index_merge = 1;`

```
mysql> set @@tidb_enable_index_merge = 1;
mysql> explain select * from t use index(idx_a, idx_b) where a > 1 or b > 1;
+-------------------------+---------+-----------+------------------------------------------------------------------+
| id                      | estRows | task      | operator info                                                    |
+-------------------------+---------+-----------+------------------------------------------------------------------+
| IndexMerge_16           | 6666.67 | root      |                                                                  |
| ├─IndexRangeScan_13     | 3333.33 | cop[tikv] | table:t, index:a, range:(1,+inf], keep order:false, stats:pseudo |
| ├─IndexRangeScan_14     | 3333.33 | cop[tikv] | table:t, index:b, range:(1,+inf], keep order:false, stats:pseudo |
| └─TableRowIDScan_15     | 6666.67 | cop[tikv] | table:t, keep order:false, stats:pseudo                          |
+-------------------------+---------+-----------+------------------------------------------------------------------+
4 rows in set (0.00 sec)
```

`IndexMerge` 使得数据库在扫描表数据时可以使用多个索引。这里 `IndexMerge_16` 算子有三个孩子结点，其中 `IndexRangeScan_13` 和 `IndexRangeScan_14` 根据范围扫描得到符合条件的所有 `RowID`，再由 `TableRowIDScan_15` 算子根据这些 `RowID` 精确的读取所有满足条件的数据。

## 如何阅读聚合的执行计划

**Hash Aggregate 示例：**

TiDB 上的 `Hash Aggregation` 算子采用多线程并发优化，执行速度快，但会消耗较多内存。下面是一个 `Hash Aggregate` 的例子：

```
TiDB(root@127.0.0.1:test) > explain select /*+ HASH_AGG() */ count(*) from t;
+---------------------------+----------+-----------+-----------------------------------------+
| id                        | estRows  | task      | operator info                           |
+---------------------------+----------+-----------+-----------------------------------------+
| HashAgg_11                | 1.00     | root      | funcs:count(Column#7)->Column#4         |
| └─TableReader_12          | 1.00     | root      | data:HashAgg_5                          |
|   └─HashAgg_5             | 1.00     | cop[tikv] | funcs:count(1)->Column#7                |
|     └─TableFullScan_8     | 10000.00 | cop[tikv] | table:t, keep order:false, stats:pseudo |
+---------------------------+----------+-----------+-----------------------------------------+
4 rows in set (0.00 sec)
```

一般而言 TiDB 的 `Hash Aggregate` 会分成两个阶段执行，一个在 TiKV/TiFlash 的 `Coprocessor` 上，计算聚合函数的中间结果。另一个在 TiDB 层，汇总所有 `Coprocessor` Task 的中间结果后，得到最终结果。

**Stream Aggregate 示例：**

TiDB `Stream Aggregation` 算子通常会比 `Hash Aggregate` 占用更少的内存，有些场景中也会比 `Hash Aggregate` 执行的更快。当数据量太大或者系统内存不足时，可以试试 `Stream Aggregate` 算子。一个 `Stream Aggregate` 的例子如下：

```
TiDB(root@127.0.0.1:test) > explain select /*+ STREAM_AGG() */ count(*) from t;
+----------------------------+----------+-----------+-----------------------------------------+
| id                         | estRows  | task      | operator info                           |
+----------------------------+----------+-----------+-----------------------------------------+
| StreamAgg_16               | 1.00     | root      | funcs:count(Column#7)->Column#4         |
| └─TableReader_17           | 1.00     | root      | data:StreamAgg_8                        |
|   └─StreamAgg_8            | 1.00     | cop[tikv] | funcs:count(1)->Column#7                |
|     └─TableFullScan_13     | 10000.00 | cop[tikv] | table:t, keep order:false, stats:pseudo |
+----------------------------+----------+-----------+-----------------------------------------+
4 rows in set (0.00 sec)
```

和 `Hash Aggregate` 类似，一般而言 TiDB 的 `Stream Aggregate` 也会分成两个阶段执行，一个在 TiKV/TiFlash 的 `Coprocessor` 上，计算聚合函数的中间结果。另一个在 TiDB 层，汇总所有 `Coprocessor` Task 的中间结果后，得到最终结果。

## 如何阅读 Join 的执行计划

TiDB 的 Join 算法包括如下几类：

- Hash Join
- Merge Join
- Index Hash Join
- Index Merge Join
- Apply

下面分别通过一些例子来解释这些 Join 算法的执行过程

**Hash Join 示例：**

TiDB 的 `Hash Join` 算子采用了多线程优化，执行速度较快，但会消耗较多内存。一个 `Hash Join` 的例子如下：

```
mysql> explain select /*+ HASH_JOIN(t1, t2) */ * from t t1 join t2 on t1.a = t2.a;
+------------------------------+----------+-----------+-------------------------------------------------------------------+
| id                           | estRows  | task      | operator info                                                     |
+------------------------------+----------+-----------+-------------------------------------------------------------------+
| HashLeftJoin_33              | 10000.00 | root      | inner join, inner:TableReader_43, equal:[eq(test.t.a, test.t2.a)] |
| ├─TableReader_43(Build)      | 10000.00 | root      | data:Selection_42                                                 |
| │ └─Selection_42             | 10000.00 | cop[tikv] | not(isnull(test.t2.a))                                            |
| │   └─TableFullScan_41       | 10000.00 | cop[tikv] | table:t2, keep order:false                                        |
| └─TableReader_37(Probe)      | 10000.00 | root      | data:Selection_36                                                 |
|   └─Selection_36             | 10000.00 | cop[tikv] | not(isnull(test.t.a))                                             |
|     └─TableFullScan_35       | 10000.00 | cop[tikv] | table:t1, keep order:false                                        |
+------------------------------+----------+-----------+-------------------------------------------------------------------+
7 rows in set (0.00 sec)
```

`Hash Join` 会将 Build 端的数据缓存在内存中，根据这些数据构造出一个 `Hash Table`，然后读取 Probe 端的数据，用 Probe 端的数据去探测（Probe）Build 端构造出来的 Hash Table，将符合条件的数据返回给用户。

**Merge Join 示例：**

TiDB 的 `Merge Join` 算子相比于 Hash Join 通常会占用更少的内存，但可能执行时间会更久。当数据量太大，或系统内存不足时，建议尝试使用。下面是一个 Merge Join 的例子：

```
mysql> explain select /*+ SM_JOIN(t1) */ * from t t1 join t t2 on t1.a = t2.a;
+------------------------------------+----------+-----------+---------------------------------------------------+
| id                                 | estRows  | task      | operator info                                     |
+------------------------------------+----------+-----------+---------------------------------------------------+
| MergeJoin_6                        | 10000.00 | root      | inner join, left key:test.t.a, right key:test.t.a |
| ├─IndexLookUp_13(Build)            | 10000.00 | root      |                                                   |
| │ ├─IndexFullScan_11(Build)        | 10000.00 | cop[tikv] | table:t2, index:a, keep order:true                |
| │ └─TableRowIDScan_12(Probe)       | 10000.00 | cop[tikv] | table:t2, keep order:false                        |
| └─IndexLookUp_10(Probe)            | 10000.00 | root      |                                                   |
|   ├─IndexFullScan_8(Build)         | 10000.00 | cop[tikv] | table:t1, index:a, keep order:true                |
|   └─TableRowIDScan_9(Probe)        | 10000.00 | cop[tikv] | table:t1, keep order:false                        |
+------------------------------------+----------+-----------+---------------------------------------------------+
7 rows in set (0.00 sec)
```

`Merge Join` 算子在执行时，会从 Build 端把一个 Join Group 的数据全部读取到内存中，接着再去读 Probe 端的数据，用 Probe 端的每行数据去和 Build 端的完整的一个 Join Group 依次去看是否匹配（除了满足等值条件以外，还有其他非等值条件，这里的 “匹配” 主要是指查看是否满足非等职条件）。Join Group 指的是所有 Join Key 上值相同的数据。

**Index Hash Join 示例：**

INL_HASH_JOIN(t1_name [, tl_name]) 提示优化器使用 Index Nested Loop Hash Join 算法。该算法与 Index Nested Loop Join 使用条件完全一样，但在某些场景下会更为节省内存资源。

```
mysql> explain select /*+ INL_HASH_JOIN(t1) */ * from t t1 join t t2 on t1.a = t2.a;
+----------------------------------+----------+-----------+---------------------------------------------------------------------------------+
| id                               | estRows  | task      | operator info                                                                   |
+----------------------------------+----------+-----------+---------------------------------------------------------------------------------+
| IndexHashJoin_32                 | 10000.00 | root      | inner join, inner:IndexLookUp_23, outer key:test.t.a, inner key:test.t.a        |
| ├─TableReader_35(Build)          | 10000.00 | root      | data:Selection_34                                                               |
| │ └─Selection_34                 | 10000.00 | cop[tikv] | not(isnull(test.t.a))                                                           |
| │   └─TableFullScan_33           | 10000.00 | cop[tikv] | table:t2, keep order:false                                                      |
| └─IndexLookUp_23(Probe)          | 1.00     | root      |                                                                                 |
|   ├─Selection_22(Build)          | 1.00     | cop[tikv] | not(isnull(test.t.a))                                                           |
|   │ └─IndexRangeScan_20          | 1.00     | cop[tikv] | table:t1, index:a, range: decided by [eq(test.t.a, test.t.a)], keep order:false |
|   └─TableRowIDScan_21(Probe)     | 1.00     | cop[tikv] | table:t1, keep order:false                                                      |
+----------------------------------+----------+-----------+---------------------------------------------------------------------------------+
8 rows in set (0.00 sec)
```

**Index Merge Join 示例：**

INL_MERGE_JOIN(t1_name [, tl_name]) 提示优化器使用 Index Nested Loop Merge Join 算法。该算法相比于 INL_JOIN 会更节省内存。该算法使用条件包含 INL_JOIN 的所有使用条件，但还需要添加一条：join keys 中的内表列集合是内表使用的 index 的前缀，或内表使用的 index 是 join keys 中的内表列集合的前缀。

```
mysql> explain select /*+ INL_MERGE_JOIN(t1) */ * from t t1 where  t1.a  in ( select t2.a from t2 where t2.b < t1.b);
+------------------------------+----------+-----------+------------------------------------------------------------------------------------------------------+
| id                           | estRows  | task      | operator info                                                                                        |
+------------------------------+----------+-----------+------------------------------------------------------------------------------------------------------+
| HashLeftJoin_26              | 8000.00  | root      | semi join, inner:TableReader_49, equal:[eq(test.t.a, test.t2.a)], other cond:lt(test.t2.b, test.t.b) |
| ├─TableReader_49(Build)      | 10000.00 | root      | data:Selection_48                                                                                    |
| │ └─Selection_48             | 10000.00 | cop[tikv] | not(isnull(test.t2.a)), not(isnull(test.t2.b))                                                       |
| │   └─TableFullScan_47       | 10000.00 | cop[tikv] | table:t2, keep order:false                                                                           |
| └─TableReader_38(Probe)      | 10000.00 | root      | data:Selection_37                                                                                    |
|   └─Selection_37             | 10000.00 | cop[tikv] | not(isnull(test.t.a)), not(isnull(test.t.b))                                                         |
|     └─TableFullScan_36       | 10000.00 | cop[tikv] | table:t1, keep order:false                                                                           |
+------------------------------+----------+-----------+------------------------------------------------------------------------------------------------------+
7 rows in set, 1 warning (0.01 sec)
```

**Apply 示例：**

```
mysql> explain select /*+ INL_MERGE_JOIN(t1) */ * from t t1 where  t1.a  in ( select avg(t2.a) from t2 where t2.b < t1.b);
+----------------------------------+----------+-----------+-------------------------------------------------------------------------------+
| id                               | estRows  | task      | operator info                                                                 |
+----------------------------------+----------+-----------+-------------------------------------------------------------------------------+
| Projection_10                    | 10000.00 | root      | test.t.id, test.t.a, test.t.b                                                 |
| └─Apply_12                       | 10000.00 | root      | semi join, inner:StreamAgg_30, equal:[eq(Column#8, Column#7)]                 |
|   ├─Projection_13(Build)         | 10000.00 | root      | test.t.id, test.t.a, test.t.b, cast(test.t.a, decimal(20,0) BINARY)->Column#8 |
|   │ └─TableReader_15             | 10000.00 | root      | data:TableFullScan_14                                                         |
|   │   └─TableFullScan_14         | 10000.00 | cop[tikv] | table:t1, keep order:false                                                    |
|   └─StreamAgg_30(Probe)          | 1.00     | root      | funcs:avg(Column#12, Column#13)->Column#7                                     |
|     └─TableReader_31             | 1.00     | root      | data:StreamAgg_19                                                             |
|       └─StreamAgg_19             | 1.00     | cop[tikv] | funcs:count(test.t2.a)->Column#12, funcs:sum(test.t2.a)->Column#13            |
|         └─Selection_29           | 8000.00  | cop[tikv] | lt(test.t2.b, test.t.b)                                                       |
|           └─TableFullScan_28     | 10000.00 | cop[tikv] | table:t2, keep order:false                                                    |
+----------------------------------+----------+-----------+-------------------------------------------------------------------------------+
10 rows in set, 1 warning (0.00 sec)
```

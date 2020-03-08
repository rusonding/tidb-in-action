常见的分页更新 SQL 一般使用主键/唯一索引进行排序，以确保相邻的两页之间没有空隙或重叠，配合 MySQL limit 语法中非常好用的 offset 功能来按固定行数拆分页面，拆分后的页面被包装在独立的事务中，可以灵活的进行逐页或批量对数据进行更新。

```
begin;
update sbtest1 set pad='new_value' where id in (select id from sbtest1 order by id limit 0,10000);
commit;
begin;
update sbtest1 set pad='new_value' where id in (select id from sbtest1 order by id limit 10000,10000);
commit;
begin;
update sbtest1 set pad='new_value' where id in (select id from sbtest1 order by id limit 20000,10000);
commit;
```
如上案例，这种方案逻辑清晰，SQL 易于编写，但它有着明显的劣势，由于需要对主键/唯一索引进行排序，越靠后的页面需要参与排序的行数越多，TiKV 中扫描数据的压力也越大，批量整体处理效率就越低，当批量的整体数据量比较大时，很可能会占用过多计算资源，甚至触发性能瓶颈，影响联机业务。
下面案例是一种改进方案，通过灵活运用窗口函数 row_number() 将数据按照主键排序后赋予行号，再通过聚合函数按照设置好的页面大小对行号进行分组，以计算出每页的最大值和最小值。

业务需求：在一小时内并发处理200万行批量数据，初始化一张表tmp_loan表结构如下

```
MySQL [demo]> desc tmp_loan;
+-------------+-------------+------+------+---------+-------+
| Field       | Type        | Null | Key  | Default | Extra |
+-------------+-------------+------+------+---------+-------+
| serialno    | int(11)     | NO   | PRI  | NULL    |       |
| name        | varchar(40) | NO   |      |         |       |
| businesssum | int(10)     | NO   |      | 0       |       |
+-------------+-------------+------+------+---------+-------+
```
初始化200万行数据

```
MySQL [demo]> select count(1) from tmp_loan;
+----------+
| count(1) |
+----------+
|  1998985 |
+----------+
MySQL [demo]> select * from tmp_loan limit 10;
+-----------+-----------+-------------+
| serialno  | name      | businesssum |
+-----------+-----------+-------------+
| 200000000 | 华碧波    |       10000 |
| 200000001 | 陶南      |       10000 |
| 200000002 | 何谷      |       10000 |
| 200000003 | 曹念      |       10000 |
| 200000004 | 潘旋千    |       10000 |
| 200000005 | 魏柔      |       10000 |
| 200000006 | 公羊      |       10000 |
| 200000007 | 司马      |       10000 |
| 200000008 | 陶之      |       10000 |
| 200000009 | 严香      |       10000 |
+-----------+-----------+-------------+
```
原来的分片方式采用 MOD 函数对主键取余，如下：
```
select serialno from tmp_table where MOD(substring(serialno,-3),${ThreadNums}) = ${ThreadId} order by serialno;
```
其中 ${ThreadNums} 是分片数量用于确定最大分片数是多少，${ThreadId} 是当前分片的序号用于确定是哪一个分片。如下：   
```
MySQL [demo]> select serialno from tmp_loan where MOD(substring(serialno,-3),17) = 1 order by serialno;
+-----------+
| serialno  |
+-----------+
| 200000001 |
| 200000018 |
| 200000035 |
| 200000052 |
| 200000069 |
| 200000086 |
| ......... |
+-----------+
117942 rows in set (1.407 sec)
```
可以看到一共分成17片，每一个分片数据量大约有12万行左右，后续通过 RabbitMQ 发送请求到批量微服务，在批量处理类中执行 sql 获取数据，采用 ResultSet 遍历结果集开始批量处理数据。
从 Mysql 切换到 TiDB 后，由于 GC lift time 设置为10min，原来采用 ResultSet 遍历结果集的方式就会由于事务时间过长而出现 GC life time is shorter than transaction duration 报错，为了避免这种异常，我们需要减少每个分片的行数以及控制 ResultSet 遍历数据集的时间，这边我们只谈减少每个分片行数，要减少行数可以通过增大总分片数量，这样可以缩短 ResultSet 遍历时间，但是通过后台监控发现同一时间运行几十条 sql 每一条都因为mod函数要整表扫描，取数时引发性能尖峰，对于联机业务会有影响。

改进方案采用窗口函数 row_number() 将数据按照主键排序后赋予行号，再通过聚合函数按照设置好的页面大小的行号进行分组，以计算书每页的最大值和最小值

```
MySQL [demo]> SELECT min(t.serialno) AS start_key, max(t.serialno) AS end_key, count(*) AS page_size FROM ( SELECT *, row_number () over (ORDER BY serialno) AS row_num FROM tmp_loan ) t GROUP BY floor((t.row_num - 1) / 50000) ORDER BY start_key;
+-----------+-----------+-----------+
| start_key | end_key   | page_size |
+-----------+-----------+-----------+
| 200000000 | 200050001 |     50000 |
| 200050002 | 200100007 |     50000 |
| 200100008 | 200150008 |     50000 |
| 200150009 | 200200013 |     50000 |
| 200200014 | 200250017 |     50000 |
|  ........ |.......... | ........  |
| 201900019 | 201950018 |     50000 |
| 201950019 | 201999003 |     48985 |
+-----------+-----------+-----------+
40 rows in set (1.51 sec)
```
可以看到这个结果集使用主键 serialno 界定好了每一分片的区间，之后只需要使用 between start_key and end_key 划分每个分片即可，使用 row_number() 函数相较于 mod 函数好处在于分片数量多的时候，减少了 TiKV 扫描数据的压力而且提高了查询速度。由于元信息的计算阶段使用主键/唯一索引进行排序，并用 row_number() 函数赋予了唯一序号，因此也可以避免在两个相邻的页面中出现空隙或重叠。
使用这种方案可以显著避免由于频繁，大量的排序造成的性能损耗，进而大幅提升批量处理的整体效率。

```
MySQL [demo]> select serialno from tmp_loan where serialno BETWEEN 200050002 and 200100007;
+-----------+
| serialno  |
+-----------+
| 200050002 |
| 200050003 |
| 200050004 |
| 200050005 |
| 200050006 |
| ......... |
+-----------+
50000 rows in set (0.070 sec)
```
当我们需要批量修改 businesssum (金额) 也可以用到窗口函数 row_number() 生成的结果集，高效的，可并发的完成数据更新：
```
MySQL [demo]> update tmp_loan set businesssum='6666' where serialno BETWEEN 200000000 and 200050001;
Query OK, 50000 rows affected (0.89 sec)
Rows matched: 50000  Changed: 50000  Warnings: 0
MySQL [demo]> select * from tmp_loan ORDER BY serialno limit 10;
+-----------+-----------+-------------+
| serialno  | name      | businesssum |
+-----------+-----------+-------------+
| 200000000 | 华碧波    |        6666 |
| 200000001 | 陶南      |        6666 |
| 200000002 | 何谷      |        6666 |
| 200000003 | 曹念      |        6666 |
| 200000004 | 潘旋千    |        6666 |
| 200000005 | 魏柔      |        6666 |
| 200000006 | 公羊      |        6666 |
| 200000007 | 司马      |        6666 |
| 200000008 | 陶之      |        6666 |
| 200000009 | 严香      |        6666 |
+-----------+-----------+-------------+
```

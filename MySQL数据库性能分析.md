# MYSQL 数据库性能问题排查

通过数据库的统计功能，从各个维度排查数据库的性能问题（版本>=5.7）。

### 一、利用Performance-Schema库的统计表，定位性能问题

- 统计哪类SQL执行最多

```sql
SELECT
    SCHEMA_NAME,
	DIGEST_TEXT,
	COUNT_STAR,
	FIRST_SEEN,
	LAST_SEEN
FROM
	performance_schema.events_statements_summary_by_digest
ORDER BY
	COUNT_STAR DESC
limit 20
```

![image-20220125224635364](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125224635364.png)



- 哪类SQL的平均响应时间最多（毫秒）

```sql
SELECT
    SCHEMA_NAME,
	DIGEST_TEXT,
	ROUND( AVG_TIMER_WAIT / 1000000000 ) as AVG_TIMER_WAIT_MS,
	COUNT_STAR
FROM
	performance_schema.events_statements_summary_by_digest
WHERE
    DIGEST_TEXT LIKE 'SELECT%'
ORDER BY
	AVG_TIMER_WAIT DESC
limit 20
```

![image-20220125224809936](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125224809936.png)



- 哪类SQL扫描记录数最多

```sql
SELECT
    SCHEMA_NAME,
	DIGEST_TEXT,
	SUM_ROWS_EXAMINED
FROM
	performance_schema.events_statements_summary_by_digest
WHERE
    DIGEST_TEXT LIKE 'SELECT%'
ORDER BY
	SUM_ROWS_EXAMINED DESC
limit 20
```

![image-20220125224921490](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125224921490.png)



- 哪个表物理IO最多

```sql
SELECT
	file_name,
	event_name,
	SUM_NUMBER_OF_BYTES_READ,
	SUM_NUMBER_OF_BYTES_WRITE
FROM
	performance_schema.file_summary_by_instance
ORDER BY
	SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE DESC
LIMIT 20
```

![image-20220125225022533](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225022533.png)



- 哪个表逻辑IO最多

```sql
SELECT
	object_name,
	COUNT_READ,
	COUNT_WRITE,
	COUNT_FETCH,
	SUM_TIMER_WAIT
FROM
	performance_schema.table_io_waits_summary_by_table
ORDER BY
	sum_timer_wait DESC
limit 20
```

![image-20220125225118778](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225118778.png)



- 哪个索引从来没有使用过

```sql
SELECT
	OBJECT_SCHEMA,
	OBJECT_NAME,
	INDEX_NAME
FROM
	performance_schema.table_io_waits_summary_by_index_usage
WHERE
	INDEX_NAME IS NOT NULL
	AND COUNT_STAR = 0
	AND OBJECT_SCHEMA <> 'mysql'
ORDER BY
	OBJECT_SCHEMA,
	OBJECT_NAME;
```

![image-20220125225318057](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225318057.png)



### 二、利用SYS库的统计表，定位性能问题

- 找出最慢的语句

```sql
SELECT 
    * 
FROM 
    sys.statements_with_runtimes_in_95th_percentile
WHERE 
	db is not null;
```

![image-20220125225453061](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225453061.png)



- 哪些语句执行了全表扫描

```sql
SELECT
	*
FROM
	sys.statements_with_full_table_scans
where
	db is not null
```

![image-20220125225552211](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225552211.png)



- 谁使用了最多的资源

```sql
# host方面
select * from sys.host_summary
```

![image-20220125225655232](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225655232.png)



```sql
# IO方面，可通过各个维度排序
select * from sys.io_global_by_file_by_bytes
```

![image-20220125225820957](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225820957.png)



```sql
# 用户方面
select * from sys.user_summary
```

![image-20220125225909361](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125225909361.png)



- 查看当前正在执行的 SQL

```sql
select conn_id, user, current_statement, last_statement from sys.session
```

![image-20220125230037337](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125230037337.png)



- 执行最多的 SQL 语句是什么

```sql
select * from sys.statement_analysis order by exec_count desc limit 10
```

![image-20220125230119308](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125230119308.png)



- 哪张表的 IO 最多？哪张表访问次数最多?

```sql
select * from sys.io_global_by_file_by_bytes limit 10
```

![image-20220125230633960](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125230633960.png)



- 哪些语句延迟比较严重?

```sql
select * from sys.statement_analysis order by avg_latency desc limit 10
```

![image-20220125230913330](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125230913330.png)



- 哪些 SQL 语句使用了磁盘临时表

```sql
select
    db, query, tmp_tables, tmp_disk_tables
from
    sys.statement_analysis
where
    tmp_tables>0 or tmp_disk_tables >0 order by (tmp_tables+tmp_disk_tables) desc limit 20
```

![image-20220125231050894](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125231050894.png)



- 哪张表占用了最多的 buffer pool

```sql
select * from sys.innodb_buffer_stats_by_table order by pages desc limit 10
```

![image-20220125231142624](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125231142624.png)



- 每个库占用多少 buffer pool

```sql
select * from sys.innodb_buffer_stats_by_schema
```

![image-20220125231226218](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125231226218.png)



- MySQL 内部现在有多个线程在运行

```sql
select user, count(*) from sys.processlist group by user
```

- 查看数据库死锁

```sql
select * from sys.innodb_lock_waits
```

​       结果：

```sql
*************************** 1. row ***************************
                wait_started: 2016-12-26 22:28:24
                    wait_age: 00:01:38
               wait_age_secs: 98
                locked_table: `test`.`test`
                locked_index: GEN_CLUST_INDEX
                 locked_type: RECORD
              waiting_trx_id: 961672
         waiting_trx_started: 2016-12-26 22:28:24
             waiting_trx_age: 00:01:38
     waiting_trx_rows_locked: 2
   waiting_trx_rows_modified: 0
                 waiting_pid: 1149284
               waiting_query: update test set id=102
             waiting_lock_id: 961672:356:3:2
           waiting_lock_mode: X
             blocking_trx_id: 961671
                blocking_pid: 1149233
              blocking_query: NULL
            blocking_lock_id: 961671:356:3:2
          blocking_lock_mode: X
        blocking_trx_started: 2016-12-26 22:25:52
            blocking_trx_age: 00:04:10
    blocking_trx_rows_locked: 1
  blocking_trx_rows_modified: 1
     sql_kill_blocking_query: KILL QUERY 1149233
sql_kill_blocking_connection: KILL 1149233
```

- 统计未使用到的索引（统计值，可能使用索引的SQL还没执行过）

```sql
select * from sys.schema_unused_indexes
```

![image-20220125231332833](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125231332833.png)



- 查找**冗余的索引**

```sql
select * from sys.schema_redundant_indexes
```

![image-20220125231408991](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125231408991.png)



- 自增ID值的最大值

```java
select * from sys.schema_auto_increment_columns;
```

![image-20220125231508549](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125231508549.png)



### 三、利用数据库慢查询日志，收集性能问题（对性能有影响，默认不开启）

- **通过命令行开启慢查询**

  - 优点：无需重启数据库，适合生产环境排查性能问题，临时开启慢查询日志的场景
  - 缺点：重启数据库后配置失效

  步骤1. 查询 slow_query_log 查看是否已开启慢查询日志：

  ```sql
  show variables like '%slow_query_log%'
  ```

  ![image-20220125161331244](C:\Users\daliu\AppData\Roaming\Typora\typora-user-images\image-20220125161331244.png)

  步骤2. 执行命令，开启慢查询:

  ```sql
  set global show_query_log='ON'
  ```

  步骤3. 指定记录慢查询日志SQL执行时间得阈值（long_query_time 单位：秒，默认10秒）

  ```sql
  set global long_query_time=3
  ```

- **通过配置my.cnf开启慢查询日志**

  步骤1. 在my.cnf文件的[mysqld]下增加如下配置开启慢查询

  ```shell
  [mysqld]
  show_query_log=ON
  long_query_time=3
  slow_query_log_file=/var/lib/mysql/localhost-slow.log
  ```

  步骤2. 重启数据库实例

- **Explain分析慢查询SQL**

  分析mysql慢查询日志 ,利用explain关键字可以模拟优化器执行SQL查询语句，来分析sql慢查询语句。举例如下：

  ```sql
  mysql> EXPLAIN select * from basic where name like '%chen%'
  +--+-----------+-----+----------+----+-------------+----+-------+----+----+--------+-----+
  |id|select_type|table|partitions|type|possible_keys| key|key_len|ref |rows|filtered|Extra|
  +--+-----------+-----+----------+----+-------------+----+-------+----+----+--------+-----+
  | 1| SIMPLE    |basic| NULL     | ALL|NULL         |NULL| NULL  |NULL|  1 | 100.00 |NULL |
  +--+-----------+-----+----------+----+-------------+----+-------+----+----+--------+-----+
  ```

  慢查询分析常用到的属性:

  1. type：访问类型
     - ALL - 常说的全表扫描
     - Index - 只遍历索引树
     - range - 只检索给定范围的行
  2. key： 实际使用的索引
     - 使用索引 -  `FORCE INDEX (index_name)`、USE INDEX (index_name)
     - 忽略索引 -  IGNORE INDEX (index_name)
  3. rows：为了找到所需的行而要读取（扫描）的行数
  4. extra：额外信息
     - Using index - 通过索引查找就能直接找到符合条件的数据，无须回表
     - Using where - 没有用到索引，回表查询
     - Using temporary - 对查询结果排序时会使用一个临时表
     - Using filesort - 对结果使用一个外部索引排序，而不是按索引次序从表里读取行
     - Using index condition - 查询的列不全在索引中，where条件中是一个前导列的范围
     - Using where;Using index - 查询的列被索引覆盖，并且where筛选条件是索引列之一，但出现了其他影响使用索引的情况（如存在范围筛选条件等）。意味着无法直接通过索引查找来查询到符合条件的数据

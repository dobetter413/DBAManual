# 快速定位锁

--- 
> FOR MySQL 8.0

## 行锁定位
```
select b.trx_mysql_thread_id as '被阻塞线程'
,b.trx_query as '被阻塞SQL'
,c.trx_mysql_thread_id as '阻塞线程'
,c.trx_query as '阻塞SQL'
,(UNIX_TIMESTAMP() - UNIX_TIMESTAMP(c.trx_started)) as '阻塞时间' 
from performance_schema.data_lock_waits a 
join information_schema.innodb_trx b
on a.REQUESTING_ENGINE_TRANSACTION_ID=b.trx_id 
join information_schema.innodb_trx c
on a.BLOCKING_ENGINE_TRANSACTION_ID=c.trx_id 
where (UNIX_TIMESTAMP() - UNIX_TIMESTAMP(c.trx_started))> 1;
```
示例
```
mysql> select b.trx_mysql_thread_id as '被阻塞线程'
    -> ,b.trx_query as '被阻塞SQL'
    -> ,c.trx_mysql_thread_id as '阻塞线程'
    -> ,c.trx_query as '阻塞SQL'
    -> ,(UNIX_TIMESTAMP() - UNIX_TIMESTAMP(c.trx_started)) as '阻塞时间' 
    -> from performance_schema.data_lock_waits a 
    -> join information_schema.innodb_trx b
    -> on a.REQUESTING_ENGINE_TRANSACTION_ID=b.trx_id 
    -> join information_schema.innodb_trx c
    -> on a.BLOCKING_ENGINE_TRANSACTION_ID=c.trx_id 
    -> where (UNIX_TIMESTAMP() - UNIX_TIMESTAMP(c.trx_started))> 1;
+-----------------+----------------------------------+--------------+-----------+--------------+
| 被阻塞线程      | 被阻塞SQL                        | 阻塞线程     | 阻塞SQL   | 阻塞时间     |
+-----------------+----------------------------------+--------------+-----------+--------------+
|              47 | delete from t_primary where id=7 |           48 | NULL      |          784 |
+-----------------+----------------------------------+--------------+-----------+--------------+
1 row in set (0.01 sec)
```


## 长事务
查看事务列表
```
select t.trx_id,t.trx_state,t.trx_started,t.trx_mysql_thread_id,t.trx_tables_in_use,to_seconds(now())-to_seconds(t.trx_started) idle_time  
from INFORMATION_SCHEMA.INNODB_TRX t 
order by idle_time desc;
返回 trx_mysql_thread_id 为 processlist_id
```
示例
```
mysql>  select t.trx_id,t.trx_state,t.trx_started,t.trx_mysql_thread_id,t.trx_tables_in_use,to_seconds(now())-to_seconds(t.trx_started) idle_time  
    -> from INFORMATION_SCHEMA.INNODB_TRX t 
    -> order by idle_time desc;
+-----------+-----------+---------------------+---------------------+-------------------+-----------+
| trx_id    | trx_state | trx_started         | trx_mysql_thread_id | trx_tables_in_use | idle_time |
+-----------+-----------+---------------------+---------------------+-------------------+-----------+
| 213480341 | RUNNING   | 2023-06-25 14:52:47 |                  48 |                 0 |      1077 |
| 213480342 | LOCK WAIT | 2023-06-25 14:53:16 |                  47 |                 1 |      1048 |
+-----------+-----------+---------------------+---------------------+-------------------+-----------+
2 rows in set (0.00 sec)
```

## 历史SQL 追踪
根据ps_id 追踪 会话执行的SQL 语句
查看历史SQL 语句
```
SELECT
  ps.id 'PROCESS ID',
  ps.USER,
  ps.HOST,
  esh.EVENT_ID,
  trx.trx_started,
  esh.event_name 'EVENT NAME',
  esh.sql_text 'SQL',
  ps.time
FROM
  performance_schema.events_statements_history esh
  JOIN performance_schema.threads th ON esh.thread_id = th.thread_id
  JOIN information_schema.PROCESSLIST ps ON ps.id = th.processlist_id
  LEFT JOIN information_schema.innodb_trx trx ON trx.trx_mysql_thread_id = ps.id
WHERE
  trx.trx_id IS NOT NULL
  AND ps.USER != 'SYSTEM_USER'
  AND ps.id in (47,48)  
ORDER BY
  esh.EVENT_ID;
```

## MDL锁
> metadata_locks是5.7中被引入，记录了metadata lock的相关信息，包括持有对象、类型、状态等信息。但5.7默认设置是关闭的（8.0默认打开），需要通过下面命令打开设置：`UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'WHERE NAME = 'wait/lock/metadata/sql/mdl';`

### 方式一：
MySQL也提供了一个类似的视图来解决metadata lock问题，视图名称为sys.schema_table_lock_waits，但此视图查询结果有bug，不是很准确。
```
mysql> select * from sys.schema_table_lock_waits;
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| object_schema | object_name | waiting_thread_id | waiting_pid | waiting_account | waiting_lock_type | waiting_lock_duration | waiting_query                                   | waiting_query_secs | waiting_query_rows_affected | waiting_query_rows_examined | blocking_thread_id | blocking_pid | blocking_account | blocking_lock_type | blocking_lock_duration | sql_kill_blocking_query | sql_kill_blocking_connection |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| cm            | t_primary   |               119 |          47 | root@127.0.0.1  | EXCLUSIVE         | TRANSACTION           | alter table t_primary add column aa varchar(20) |                116 |                           0 |                           0 |                119 |           47 | root@127.0.0.1   | SHARED_UPGRADABLE  | TRANSACTION            | KILL QUERY 47           | KILL 47                      |
| cm            | t_primary   |               119 |          47 | root@127.0.0.1  | EXCLUSIVE         | TRANSACTION           | alter table t_primary add column aa varchar(20) |                116 |                           0 |                           0 |                120 |           48 | root@127.0.0.1   | SHARED_WRITE       | TRANSACTION            | KILL QUERY 48           | KILL 48                      |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
2 rows in set (0.00 sec)
```
> 注意，waiting_pid=blocking_pid 说明这个是被阻塞的，kill 这个pid 是无法解决锁源头的，其blocking_lock_type 为SHARED_UPGRADABLE。

> 可以使用如下SQL 进行筛选
```
select * from sys.schema_table_lock_waits where waiting_pid!=blocking_pid;
```
示例：
```
mysql> select * from sys.schema_table_lock_waits where waiting_pid!=blocking_pid;
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| object_schema | object_name | waiting_thread_id | waiting_pid | waiting_account | waiting_lock_type | waiting_lock_duration | waiting_query                                   | waiting_query_secs | waiting_query_rows_affected | waiting_query_rows_examined | blocking_thread_id | blocking_pid | blocking_account | blocking_lock_type | blocking_lock_duration | sql_kill_blocking_query | sql_kill_blocking_connection |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| cm            | t_primary   |               119 |          47 | root@127.0.0.1  | EXCLUSIVE         | TRANSACTION           | alter table t_primary add column aa varchar(20) |                337 |                           0 |                           0 |                120 |           48 | root@127.0.0.1   | SHARED_WRITE       | TRANSACTION            | KILL QUERY 48           | KILL 48                      |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
1 row in set (0.00 sec)
```

### 方式二：
>此SQL 针对DML 事务未提交，阻塞DDL 导致MDL 锁的场景进行排查,查询出阻塞DDL 的DML
```
SELECT locked_schema,
locked_table,
locked_type,
waiting_processlist_id,
waiting_time,
waiting_query,
waiting_state,
blocking_processlist_id,
blocking_time,
substring_index(sql_text,"transaction_begin;" ,-1) AS blocking_query,
sql_kill_blocking_connection
FROM 
( 
SELECT 
b.OWNER_THREAD_ID AS granted_thread_id,
a.OBJECT_SCHEMA AS locked_schema,
a.OBJECT_NAME AS locked_table,
"Metadata Lock" AS locked_type,
c.PROCESSLIST_ID AS waiting_processlist_id,
c.PROCESSLIST_TIME AS waiting_time,
c.PROCESSLIST_INFO AS waiting_query,
c.PROCESSLIST_STATE AS waiting_state,
d.PROCESSLIST_ID AS blocking_processlist_id,
d.PROCESSLIST_TIME AS blocking_time,
d.PROCESSLIST_INFO AS blocking_query,
concat('KILL ', d.PROCESSLIST_ID) AS sql_kill_blocking_connection
FROM performance_schema.metadata_locks a JOIN performance_schema.metadata_locks b ON a.OBJECT_SCHEMA = b.OBJECT_SCHEMA AND a.OBJECT_NAME = b.OBJECT_NAME
AND a.lock_status = 'PENDING'
AND b.lock_status = 'GRANTED'
AND a.OWNER_THREAD_ID <> b.OWNER_THREAD_ID
AND a.lock_type = 'EXCLUSIVE'
JOIN performance_schema.threads c ON a.OWNER_THREAD_ID = c.THREAD_ID JOIN performance_schema.threads d ON b.OWNER_THREAD_ID = d.THREAD_ID
) t1,
(
SELECT thread_id, group_concat( CASE WHEN EVENT_NAME = 'statement/sql/begin' THEN "transaction_begin" ELSE sql_text END ORDER BY event_id SEPARATOR ";" ) AS sql_text
FROM
performance_schema.events_statements_history
GROUP BY thread_id
) t2
WHERE t1.granted_thread_id = t2.thread_id \G
```

--- 
>FOR MySQL 5.7

## 行锁定位
```
select b.trx_mysql_thread_id as '被阻塞线程ps_id',
b.trx_query as '被阻塞SQL',
b.trx_started as '被阻塞事务开始时间',
c.trx_mysql_thread_id as '阻塞线程ps_id',
c.trx_query as  '阻塞SQL',
c.trx_started as '阻塞事务开始时间',
(UNIX_TIMESTAMP() - UNIX_TIMESTAMP(c.trx_started)) as '阻塞事务持续时长',
(UNIX_TIMESTAMP() - UNIX_TIMESTAMP(b.trx_wait_started)) as '被阻塞事务受阻时长',
concat('kill ',c.trx_mysql_thread_id,';') as KILL_QUERY
from information_schema.INNODB_LOCK_WAITS a 
join information_schema.INNODB_TRX b on a.requesting_trx_id=b.trx_id
join information_schema.INNODB_TRX c on a.blocking_trx_id=c.trx_id
where (UNIX_TIMESTAMP() - UNIX_TIMESTAMP(b.trx_wait_started)) >1\G
```

## 长事务
```
select t.trx_id,t.trx_state,t.trx_started,t.trx_mysql_thread_id,t.trx_tables_in_use,to_seconds(now())-to_seconds(t.trx_started) idle_time  
from INFORMATION_SCHEMA.INNODB_TRX t 
order by idle_time desc;
返回 trx_mysql_thread_id 为 processlist_id
```


## 历史SQL 追踪
根据ps_id 追踪 会话执行的SQL 语句
查看历史SQL 语句
```
SELECT
  ps.id 'PROCESS ID',
  ps.USER,
  ps.HOST,
  esh.EVENT_ID,
  trx.trx_started,
  esh.event_name 'EVENT NAME',
  esh.sql_text 'SQL',
  ps.time
FROM
  performance_schema.events_statements_history esh
  JOIN performance_schema.threads th ON esh.thread_id = th.thread_id
  JOIN information_schema.PROCESSLIST ps ON ps.id = th.processlist_id
  LEFT JOIN information_schema.innodb_trx trx ON trx.trx_mysql_thread_id = ps.id
WHERE
  trx.trx_id IS NOT NULL
  AND ps.USER != 'SYSTEM_USER'
  AND ps.id in (14,15)  
ORDER BY
  esh.EVENT_ID;
```

## MDL 锁
> metadata_locks是5.7中被引入，记录了metadata lock的相关信息，包括持有对象、类型、状态等信息。但5.7默认设置是关闭的（8.0默认打开），需要通过下面命令打开设置：`UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'WHERE NAME = 'wait/lock/metadata/sql/mdl';`

### 方式一：
MySQL也提供了一个类似的视图来解决metadata lock问题，视图名称为sys.schema_table_lock_waits，但此视图查询结果有bug，不是很准确。
```
mysql> select * from sys.schema_table_lock_waits;
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| object_schema | object_name | waiting_thread_id | waiting_pid | waiting_account | waiting_lock_type | waiting_lock_duration | waiting_query                                   | waiting_query_secs | waiting_query_rows_affected | waiting_query_rows_examined | blocking_thread_id | blocking_pid | blocking_account | blocking_lock_type | blocking_lock_duration | sql_kill_blocking_query | sql_kill_blocking_connection |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| cm            | t_primary   |               119 |          47 | root@127.0.0.1  | EXCLUSIVE         | TRANSACTION           | alter table t_primary add column aa varchar(20) |                116 |                           0 |                           0 |                119 |           47 | root@127.0.0.1   | SHARED_UPGRADABLE  | TRANSACTION            | KILL QUERY 47           | KILL 47                      |
| cm            | t_primary   |               119 |          47 | root@127.0.0.1  | EXCLUSIVE         | TRANSACTION           | alter table t_primary add column aa varchar(20) |                116 |                           0 |                           0 |                120 |           48 | root@127.0.0.1   | SHARED_WRITE       | TRANSACTION            | KILL QUERY 48           | KILL 48                      |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
2 rows in set (0.00 sec)
```
> 注意，waiting_pid=blocking_pid 说明这个是被阻塞的，kill 这个pid 是无法解决锁源头的，其blocking_lock_type 为SHARED_UPGRADABLE。

> 可以使用如下SQL 进行筛选
```
select * from sys.schema_table_lock_waits where waiting_pid!=blocking_pid;
```
示例：
```
mysql> select * from sys.schema_table_lock_waits where waiting_pid!=blocking_pid;
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| object_schema | object_name | waiting_thread_id | waiting_pid | waiting_account | waiting_lock_type | waiting_lock_duration | waiting_query                                   | waiting_query_secs | waiting_query_rows_affected | waiting_query_rows_examined | blocking_thread_id | blocking_pid | blocking_account | blocking_lock_type | blocking_lock_duration | sql_kill_blocking_query | sql_kill_blocking_connection |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
| cm            | t_primary   |               119 |          47 | root@127.0.0.1  | EXCLUSIVE         | TRANSACTION           | alter table t_primary add column aa varchar(20) |                337 |                           0 |                           0 |                120 |           48 | root@127.0.0.1   | SHARED_WRITE       | TRANSACTION            | KILL QUERY 48           | KILL 48                      |
+---------------+-------------+-------------------+-------------+-----------------+-------------------+-----------------------+-------------------------------------------------+--------------------+-----------------------------+-----------------------------+--------------------+--------------+------------------+--------------------+------------------------+-------------------------+------------------------------+
1 row in set (0.00 sec)
```
### 方式二
>此SQL 针对DML 事务未提交，阻塞DDL 导致MDL 锁的场景进行排查,查询出阻塞DDL 的DML
```
SELECT locked_schema,
locked_table,
locked_type,
waiting_processlist_id,
waiting_time,
waiting_query,
waiting_state,
blocking_processlist_id,
blocking_time,
substring_index(sql_text,"transaction_begin;" ,-1) AS blocking_query,
sql_kill_blocking_connection
FROM 
( 
SELECT 
b.OWNER_THREAD_ID AS granted_thread_id,
a.OBJECT_SCHEMA AS locked_schema,
a.OBJECT_NAME AS locked_table,
"Metadata Lock" AS locked_type,
c.PROCESSLIST_ID AS waiting_processlist_id,
c.PROCESSLIST_TIME AS waiting_time,
c.PROCESSLIST_INFO AS waiting_query,
c.PROCESSLIST_STATE AS waiting_state,
d.PROCESSLIST_ID AS blocking_processlist_id,
d.PROCESSLIST_TIME AS blocking_time,
d.PROCESSLIST_INFO AS blocking_query,
concat('KILL ', d.PROCESSLIST_ID) AS sql_kill_blocking_connection
FROM performance_schema.metadata_locks a JOIN performance_schema.metadata_locks b ON a.OBJECT_SCHEMA = b.OBJECT_SCHEMA AND a.OBJECT_NAME = b.OBJECT_NAME
AND a.lock_status = 'PENDING'
AND b.lock_status = 'GRANTED'
AND a.OWNER_THREAD_ID <> b.OWNER_THREAD_ID
AND a.lock_type = 'EXCLUSIVE'
JOIN performance_schema.threads c ON a.OWNER_THREAD_ID = c.THREAD_ID JOIN performance_schema.threads d ON b.OWNER_THREAD_ID = d.THREAD_ID
) t1,
(
SELECT thread_id, group_concat( CASE WHEN EVENT_NAME = 'statement/sql/begin' THEN "transaction_begin" ELSE sql_text END ORDER BY event_id SEPARATOR ";" ) AS sql_text
FROM
performance_schema.events_statements_history
GROUP BY thread_id
) t2
WHERE t1.granted_thread_id = t2.thread_id \G
```
---

## 附录
+ 历史SQL 开关

```
表级的设置
select * from performance_schema.setup_consumers where name like '%events_statements_history_long%';

字段级的设置
SELECT * FROM performance_schema.setup_instruments where name like '%events_statements_history_long%';
```

+ MDL 开关

metadata_locks是5.7中被引入，记录了metadata lock的相关信息，包括持有对象、类型、状态等信息。但5.7默认设置是关闭的（8.0默认打开），需要通过下面命令打开设置

```
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'WHERE NAME = 'wait/lock/metadata/sql/mdl';
```
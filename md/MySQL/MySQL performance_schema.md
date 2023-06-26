# Performance_schema

## 介绍
performance_schema 用于监控MySQL server在一个较低级别的运行过程中的资源消耗、资源等待等情况。只存在于内存中，不记录binlog、不参与复制
performance_schema 无法监控其自身的资源消耗情况

## 配置
1. 开关
   show variables like '%performance_schema%'; 查看是否开始performance_schema
   在配置文件中启用 performance_schema
   ```
   [mysqld]
    performance_schema=ON
   ```
2. 生产者和消费者
    + instruments: 生产者，用于采集mysql中各种各样的操作产生的事件信息，对应配置表中的配置项我们可以称为监控采集配置项。
    + consumers:消费者，对应的消费者表用于存储来自instruments采集的数据，对应配置表中的配置项我们可以称为消费存储配置项。
3. 表分类
   ```
    --语句事件记录表，这些表记录了语句事件信息，当前语句事件表events_statements_current、历史语句事件表events_statements_history和长语句历史事件表events_statements_history_long、以及聚合后的摘要表summary，其中，summary表还可以根据帐号(account)，主机(host)，程序(program)，线程(thread)，用户(user)和全局(global)再进行细分)
    show tables like '%statement%';
    ​
    --等待事件记录表，与语句事件类型的相关记录表类似：
    show tables like '%wait%';
    ​
    --阶段事件记录表，记录语句执行的阶段事件的表
    show tables like '%stage%';
    ​
    --事务事件记录表，记录事务相关的事件的表
    show tables like '%transaction%';
    ​
    --监控文件系统层调用的表
    show tables like '%file%';
    ​
    --监视内存使用的表
    show tables like '%memory%';
    ​
    --动态对performance_schema进行配置的配置表
    show tables like '%setup%';
   ```
4. 配置与使用
   ```
   --打开等待事件的采集器配置项开关，需要修改setup_instruments配置表中对应的采集器配置项
    UPDATE setup_instruments SET ENABLED = 'YES', TIMED = 'YES'where name like 'wait%';
    ​
    --打开等待事件的保存表配置开关，修改setup_consumers配置表中对应的配置项
    UPDATE setup_consumers SET ENABLED = 'YES'where name like '%wait%';
    ​
    --当配置完成之后可以查看当前server正在做什么，可以通过查询events_waits_current表来得知，该表中每个线程只包含一行数据，用于显示每个线程的最新监视事件
    select * from events_waits_current\G
    *************************** 1. row ***************************
                THREAD_ID: 11
                 EVENT_ID: 570
             END_EVENT_ID: 570
               EVENT_NAME: wait/synch/mutex/innodb/buf_dblwr_mutex
                   SOURCE: 
              TIMER_START: 4508505105239280
                TIMER_END: 4508505105270160
               TIMER_WAIT: 30880
                    SPINS: NULL
            OBJECT_SCHEMA: NULL
              OBJECT_NAME: NULL
               INDEX_NAME: NULL
              OBJECT_TYPE: NULL
    OBJECT_INSTANCE_BEGIN: 67918392
         NESTING_EVENT_ID: NULL
       NESTING_EVENT_TYPE: NULL
                OPERATION: lock
          NUMBER_OF_BYTES: NULL
                    FLAGS: NULL
    /*该信息表示线程id为11的线程正在等待buf_dblwr_mutex锁，等待事件为30880
    属性说明：
      id:事件来自哪个线程，事件编号是多少
      event_name:表示检测到的具体的内容
      source:表示这个检测代码在哪个源文件中以及行号
      timer_start:表示该事件的开始时间
      timer_end:表示该事件的结束时间
      timer_wait:表示该事件总的花费时间
    注意：_current表中每个线程只保留一条记录，一旦线程完成工作，该表中不会再记录该线程的事件信息
    */
    ​
    /*
    _history表中记录每个线程应该执行完成的事件信息，但每个线程的事件信息只会记录10条，再多就会被覆盖，*_history_long表中记录所有线程的事件信息，但总记录数量是10000，   超过就会被覆盖掉
    */
    select thread_id,event_id,event_name,timer_wait from events_waits_history order by thread_id limit 21;
    ​
    /*
    summary表提供所有事件的汇总信息，该组中的表以不同的方式汇总事件数据（如：按用户，按主机，按线程等等）。例如：要查看哪些instruments占用最多的时间，可以通过对    events_waits_summary_global_by_event_name表的COUNT_STAR或SUM_TIMER_WAIT列进行查询（这两列是对事件的记录数执行COUNT（*）、事件记录的TIMER_WAIT列执行SUM  （TIMER_WAIT）统计而来）
    */
    SELECT EVENT_NAME,COUNT_STAR FROM events_waits_summary_global_by_event_name  ORDER BY COUNT_STAR DESC LIMIT 10;
    ​
    /*
    instance表记录了哪些类型的对象会被检测。这些对象在被server使用时，在该表中将会产生一条事件记录，例如，file_instances表列出了文件I/O操作及其关联文件名
    */
    select * from file_instances limit 20; 
       ```
5. 常用配置
 + 启动选项
```
performance_schema_consumer_events_statements_current=TRUE
是否在mysql server启动时就开启events_statements_current表的记录功能(该表记录当前的语句事件信息)，启动之后也可以在setup_consumers表中使用UPDATE语句进动态  更新setup_consumers配置表中的events_statements_current配置项，默认值为TRUE
​
performance_schema_consumer_events_statements_history=TRUE
与performance_schema_consumer_events_statements_current选项类似，但该选项是用于配置是否记录语句事件短历史信息，默认为TRUE
​
performance_schema_consumer_events_stages_history_long=FALSE
与performance_schema_consumer_events_statements_current选项类似，但该选项是用于配置是否记录语句事件长历史信息，默认为FALSE
​
除了statement(语句)事件之外，还支持：wait(等待)事件、state(阶段)事件、transaction(事务)事件，他们与statement事件一样都有三个启动项分别进行配置，但这些待事件  默认未启用，如果需要在MySQL Server启动时一同启动，则通常需要写进my.cnf配置文件中
performance_schema_consumer_global_instrumentation=TRUE
是否在MySQL Server启动时就开启全局表（如：mutex_instances、rwlock_instances、cond_instances、file_instances、users、hostsaccounts、socket_summary_by_event_name、file_summary_by_instance等大部分的全局对象计数统计和事件汇总统计信息表 ）的记录功能，启动之后也可以在setup_consumers中使 用UPDATE语句进行动态更新全局配置项
默认值为TRUE
​
performance_schema_consumer_statements_digest=TRUE
是否在MySQL Server启动时就开启events_statements_summary_by_digest 表的记录功能，启动之后也可以在setup_consumers表中使用UPDATE语句进行动态更新diges配置 项
默认值为TRUE
​
performance_schema_consumer_thread_instrumentation=TRUE
是否在MySQL Server启动时就开启
​
events_xxx_summary_by_yyy_by_event_name表的记录功能，启动之后也可以在setup_consumers表中使用UPDATE语句进行动态更新线程配置项
默认值为TRUE
​
performance_schema_instrument[=name]
是否在MySQL Server启动时就启用某些采集器，由于instruments配置项多达数千个，所以该配置项支持key-value模式，还支持%号进行通配等，如下:
​
# [=name]可以指定为具体的Instruments名称（但是这样如果有多个需要指定的时候，就需要使用该选项多次），也可以使用通配符，可以指定instruments相同的前缀+通配符也可    以使用%代表所有的instruments
​
## 指定开启单个instruments
​
--performance-schema-instrument= 'instrument_name=value'
​
## 使用通配符指定开启多个instruments
​
--performance-schema-instrument= 'wait/synch/cond/%=COUNTED'
​
## 开关所有的instruments
​
--performance-schema-instrument= '%=ON'
​
--performance-schema-instrument= '%=OFF'
```
> 注意，这些启动选项要生效的前提是，需要设置performance_schema=ON。另外，这些启动选项虽然无法使用show variables语句查看，但我们可以通setup_instruments和setup_consumers表查询这些选项指定的值。

 + 系统变量
```
show variables like '%performance_schema%';
--重要的属性解释
performance_schema=ON
/*
控制performance_schema功能的开关，要使用MySQL的performance_schema，需要在mysqld启动时启用，以启用事件收集功能
该参数在5.7.x之前支持performance_schema的版本中默认关闭，5.7.x版本开始默认开启
注意：如果mysqld在初始化performance_schema时发现无法分配任何相关的内部缓冲区，则performance_schema将自动禁用，并将performance_schema设置为OFF
*/
​
performance_schema_digests_size=10000
/*
控制events_statements_summary_by_digest表中的最大行数。如果产生的语句摘要信息超过此最大值，便无法继续存入该表，此时performance_schema会增加状态变量
*/
performance_schema_events_statements_history_long_size=10000
/*
控制events_statements_history_long表中的最大行数，该参数控制所有会话在events_statements_history_long表中能够存放的总事件记录数，超过这个限制之后，最的记  录将被覆盖
全局变量，只读变量，整型值，5.6.3版本引入 * 5.6.x版本中，5.6.5及其之前的版本默认为10000，5.6.6及其之后的版本默认值为-1，通常情况下，自动计算的值都是10000* 5.  7.x版本中，默认值为-1，通常情况下，自动计算的值都是10000
*/
performance_schema_events_statements_history_size=10
/*
控制events_statements_history表中单个线程（会话）的最大行数，该参数控制单个会话在events_statements_history表中能够存放的事件记录数，超过这个限制之后，个会    话最早的记录将被覆盖
全局变量，只读变量，整型值，5.6.3版本引入 * 5.6.x版本中，5.6.5及其之前的版本默认为10，5.6.6及其之后的版本默认值为-1，通常情况下，自动计算的值都是10 * 5.7.版本 中，默认值为-1，通常情况下，自动计算的值都是10
除了statement(语句)事件之外，wait(等待)事件、state(阶段)事件、transaction(事务)事件，他们与statement事件一样都有三个参数分别进行存储限制配置，有兴趣的学自行  研究，这里不再赘述
*/
performance_schema_max_digest_length=1024
/*
用于控制标准化形式的SQL语句文本在存入performance_schema时的限制长度，该变量与max_digest_length变量相关(max_digest_length变量含义请自行查阅相关资料)
全局变量，只读变量，默认值1024字节，整型值，取值范围0~1048576
*/
performance_schema_max_sql_text_length=1024
/*
控制存入events_statements_current，events_statements_history和events_statements_history_long语句事件表中的SQL_TEXT列的最大SQL长度字节数。 超系统变    量performance_schema_max_sql_text_length的部分将被丢弃，不会记录，一般情况下不需要调整该参数，除非被截断的部分与其他SQL比起来有很大差异
全局变量，只读变量，整型值，默认值为1024字节，取值范围为0~1048576，5.7.6版本引入
降低系统变量performance_schema_max_sql_text_length值可以减少内存使用，但如果汇总的SQL中，被截断部分有较大差异，会导致没有办法再对这些有较大差异的SQL进区  分。 增加该系统变量值会增加内存使用，但对于汇总SQL来讲可以更精准地区分不同的部分。
*/
```



## 常用语句
```
--1、哪类的SQL执行最多？
SELECT DIGEST_TEXT,COUNT_STAR,FIRST_SEEN,LAST_SEEN FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC
--2、哪类SQL的平均响应时间最多？
SELECT DIGEST_TEXT,AVG_TIMER_WAIT FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC
--3、哪类SQL排序记录数最多？
SELECT DIGEST_TEXT,SUM_SORT_ROWS FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC
--4、哪类SQL扫描记录数最多？
SELECT DIGEST_TEXT,SUM_ROWS_EXAMINED FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC
--5、哪类SQL使用临时表最多？
SELECT DIGEST_TEXT,SUM_CREATED_TMP_TABLES,SUM_CREATED_TMP_DISK_TABLES FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC
--6、哪类SQL返回结果集最多？
SELECT DIGEST_TEXT,SUM_ROWS_SENT FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC
--7、哪个表物理IO最多？
SELECT file_name,event_name,SUM_NUMBER_OF_BYTES_READ,SUM_NUMBER_OF_BYTES_WRITE FROM file_summary_by_instance ORDER BY  SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE DESC
--8、哪个表逻辑IO最多？
SELECT object_name,COUNT_READ,COUNT_WRITE,COUNT_FETCH,SUM_TIMER_WAIT FROM table_io_waits_summary_by_table ORDER BY sum_timer_wait DESC
--9、哪个索引访问最多？
SELECT OBJECT_NAME,INDEX_NAME,COUNT_FETCH,COUNT_INSERT,COUNT_UPDATE,COUNT_DELETE FROM table_io_waits_summary_by_index_usage ORDERBY    SUM_TIMER_WAIT DESC
--10、哪个索引从来没有用过？
SELECT OBJECT_SCHEMA,OBJECT_NAME,INDEX_NAME FROM table_io_waits_summary_by_index_usage WHERE INDEX_NAME IS NOT NULL AND COUNT_STAR = 0AND  OBJECT_SCHEMA <> 'mysql' ORDER BY OBJECT_SCHEMA,OBJECT_NAME;
--11、哪个等待事件消耗时间最多？
SELECT EVENT_NAME,COUNT_STAR,SUM_TIMER_WAIT,AVG_TIMER_WAIT FROM events_waits_summary_global_by_event_name WHERE event_name != 'idle'ORDER  BY SUM_TIMER_WAIT DESC
--12-1、剖析某条SQL的执行情况，包括statement信息，stege信息，wait信息
SELECT EVENT_ID,sql_text FROM events_statements_history WHERE sql_text LIKE '%count(*)%';
--12-2、查看每个阶段的时间消耗
SELECT event_id,EVENT_NAME,SOURCE,TIMER_END - TIMER_START FROM events_stages_history_long WHERE NESTING_EVENT_ID = 1553;
--12-3、查看每个阶段的锁等待情况
SELECT event_id,event_name,source,timer_wait,object_name,index_name,operation,nesting_event_id FROM events_waits_history_longWHERE nesting_event_id = 1553;
```

## 附录
参考链接
https://blog.csdn.net/f746262041/article/details/123548671
https://blog.csdn.net/f746262041/article/details/123689963
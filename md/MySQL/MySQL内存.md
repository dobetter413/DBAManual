#MySQL 内存占用情况分析

## 观察内存消耗
1. 查看进程号
`ps -ef|grep mysqld`
2. 查看指定进程top 信息
`top -p pid_of_mysqld`
3. 查看指定进程及其线程top 信息
`top -H -p pid_of_mysqld`

> top 下按 'e' 切换内存显示单位


## 内存消耗点
1. 全局共享内存
    + innodb_buffer_pool_size：InnoDB缓冲池的大小
    + innodb_log_buffer_size：InnoDB日志缓冲的大小
    + key_buffer_size：MyISAM缓存索引块的内存大小
    + query_cache_size：查询缓冲的大小，8.0已被移除
  
2. 线程独占内存
    + thread_stack：每个线程分配的堆栈大小
    + sort_buffer_size：排序缓冲的大小
    + join_buffer_size：连接缓冲的大小
    + read_buffer_size：MyISAM顺序读缓冲的大小
    + read_rnd_buffer_size：MyISAM随机读缓冲的大小、MRR缓冲的大小
    + tmp_table_size/max_heap_table_size：内存临时表的大小
    + binlog_cache_size：二进制日志缓冲的大小
  
3. 其他，程序运行也会消耗一些内存
   

## 内存分配器
> 在MySQL中，buffer pool的内存，是通过mmap()方式直接向操作系统申请分配；除此之外，大多数的内存管理，都需要经过内存分配器。为了实现更高效的内存管理，避免频繁的内存分配与回收，内存分配器会长时间占用大量内存，以供内部重复使用。关于内存分配器的选择，推荐使用jemalloc，可以有效解决内存碎片与提升整体性能。
> 因此，MySQL占用内存高的原因可能包括：innodb_buffer_pool_size设置过大、连接数/并发数过高、大量排序操作、内存分配器占用、以及MySQL Bug等等。一般来说，在MySQL整个运行周期内，刚启动时内存上涨会比较快，运行一段时间后会逐渐趋于平稳，这种情况是不需要过多关注的；如果在稳定运行后，出现内存突增、内存持续增长不释放的情况，那就需要我们进一步分析是什么原因造成的。


## performance_schema
1. 查看采集项开关
`SELECT * FROM performance_schema.setup_instruments  WHERE NAME LIKE '%memory/%';`

2. 开启采集项开关
`update performance_schema.setup_instruments set enabled = 'yes' where name like 'memory%';`

3. 监控维度
用于从各维度查看内存的消耗：

   用户维度对内存进行监控    memory_summary_by_account_by_event_name
   主机维度对内存进行监控    memory_summary_by_host_by_event_name
   线程维度对内存进行监控    memory_summary_by_thread_by_event_name
   账号维度对内存进行监控    memory_summary_by_user_by_event_name
   全局维度对内存进行监控    memory_summary_global_by_event_name

4. 统计内存消耗

+ 统计事件消耗内存,那个event_name排最前,就代表这个事件模块占内存最多.例如:JOIN_CACHE就是join的操作,mem0mem就是客户端连接,没开启统计的话,数据可能并不会很多.

```
select event_name, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_global_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select event_name, CURRENT_NUMBER_OF_BYTES_USED/1024/1024 as CURRENT_NUMBER_MiB_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_global_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select sum(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024 as SUM_CURRENT_MiB_USED from performance_schema.memory_summary_global_by_event_name limit 10;
```
+ 统计线程消耗内存,thread_id就是show processlist的thread_id,如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计
  
```
select thread_id, event_name, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_thread_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select thread_id, event_name, CURRENT_NUMBER_OF_BYTES_USED/1024/1024 as CURRENT_NUMBER_MiB_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_thread_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select sum(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024 as SUM_CURRENT_MiB_USED from performance_schema.memory_summary_by_thread_by_event_name limit 10;
```

+ 统计账户消耗内存,如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计

```
select USER, HOST, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_account_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select USER, HOST, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED/1024/1024 as CURRENT_NUMBER_MiB_USED ,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_account_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select sum(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024 as SUM_CURRENT_NUMBER_MiB_USED  from performance_schema.memory_summary_by_account_by_event_name ;
```

+ 统计主机消耗内存,如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计

```
select HOST, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_host_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select HOST, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED/1024/1024 as CURRENT_NUMBER_MiB_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_host_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select sum(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024 as SUM_CURRENT_NUMBER_MiB_USED from performance_schema.memory_summary_by_host_by_event_name;
```

+ 统计用户消耗内存,如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计

```
select USER, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_user_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;

select USER, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED/1024/1024 as CURRENT_NUMBER_MiB_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_user_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10; 

select sum(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024 as SUM_CURRENT_NUMBER_MiB_USED from performance_schema.memory_summary_by_user_by_event_name ;
```

## sys schema
1. 使用sys 统计内存使用
+ 按表级别对象维度查看innodb的内存分布状况
`select * from sys.innodb_buffer_stats_by_table;`
+ 按schema维度查看innodb的内存分布状况
`select * from sys.innodb_buffer_stats_by_schema;`
+ 按照线程维度:
`select thread_id,user,current_allocated from sys.memory_by_thread_by_current_bytes;`
+ 按照用户维度:
`select user,current_allocated from sys.memory_by_user_by_current_bytes;`
+ 按照客户端主机
`select host,current_allocated from sys.memory_by_host_by_current_bytes;`
+ 按照事件类型
`select event_name,current_alloc,high_count from sys.memory_global_by_current_bytes;`




理想情况下，最大内存占用量计算公式
```
select  ifnull(@@key_buffer_size,0) 
      + ifnull(@@innodb_buffer_pool_size,0)+ifnull(@@innodb_log_buffer_size,0)
      + ifnull(@@max_connections,0) *(
           ifnull(@@read_buffer_size,0) + ifnull(@@read_rnd_buffer_size,0)
        + ifnull(@@sort_buffer_size,0) + ifnull(@@join_buffer_size,0)
        + ifnull(@@binlog_cache_size,0) + ifnull(@@thread_stack,0) + ifnull(@@tmp_table_size,0))
```

>理论总内存耗用依次为：MyISAM存储引擎占用量、内部临时表、innodb_buffer_pool、redolog缓冲，再加上连接相关的内存占用：表扫描、无序读、排序、JOIN、binlog缓冲、线程栈。



## 分析案例
gdb 分析glibc 内存情况

`gdb -ex "call (void) malloc_stats()" --batch -p 12807`

`awk '{if($1 == "system") total+=$NF; else if ($1 == "in") used+=$NF }END{print total/1024/1024/1024,used/1024/1024/1024}' ./fx.txt`
0.552795 0.49119
分配给 MySQL 内存0.55G，MySQL 使用内存 0.49G，如果差距较大则说明碎片化严重，需要考虑更换内存分配器

是否使用swap
`cat /proc/$(pidof mysqld)/status | grep Swap`




## 分析案例
https://www.modb.pro/db/512361

https://blog.csdn.net/liuhuayang/article/details/114529025

## 内存分配

glibc 采用 ptmalloc2 作为内存分配器，有个缺点就是用户free掉的内存不会马上归还操作系统，有ptmalloc统一管理空闲的块，避免频繁的系统调用

## 清理glibc 内存碎片

`gdb --batch --pid 21680 --ex 'call (void) malloc_trim(0)'`

## 安装 jemalloc 
```
./configure --prefix=/usr/local/jemalloc --libdir=/usr/lib
make && make install
echo /usr/lib >> /etc/ld.so.conf
ldconfig

[mysqld_safe]
malloc-lib=/usr/lib/libjemalloc.so
```
启动mysql 验证是否生效
`lsof -n | grep libjemalloc.so`


## MySQL 内存分配器选型测试

> 对glibc、jemalloc、tcmalloc 三种内存分配器的对比
https://www.jianshu.com/p/d6f327af62f4



## ARM 环境 jemalloc 相关问题
https://m.elecfans.com/article/1832102.html

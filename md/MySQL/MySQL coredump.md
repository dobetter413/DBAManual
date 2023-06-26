# MySQL coredump
> MySQL coredump 处理流程

>程序运行过程中可能会异常终止或崩溃，OS会把程序挂掉时的内存状态记录下来，写入core文件，这就叫 coredump，通过gdb结合core文件可以方便地进行调试。利用core文件中保留的异常堆栈文件，能够帮助研发同学更快定位问题。因此，如果某些故障断断续续会出现，建议阶段性开启coredump功能。

## 启用coredump
1. 系统配置
```
$ ulimit -c unlimited
$ sysctl -w fs.suid_dumpable=2
$ echo "core.%p.%e.%s" > /proc/sys/kernel/core_pattern
```
同时，将这些修改持久化到相应文件中（假定MySQL/GreatSQL服务进程的属主用户是  mysql）
```
$ echo "mysql  -  core   unlimited" >> /etc/security/limits.conf
$ echo "fs.suid_dumpable=2" >> /etc/sysctl.conf
$ echo "kernel.core_pattern=core.%e.%p.%t" >> /etc/sysctl.conf
$ sysctl -p
```

2. 数据库配置
在mysql配置文件添加如下参数开始coredump 
```
core_file
innodb_buffer_pool_in_core_file=OFF
```
以上两个参数重启生效

参数查看
```
mysql> show global variables like '%core%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| core_file                       | ON    |
| innodb_buffer_pool_in_core_file | OFF   |
+---------------------------------+-------+
```

3. 模拟coredump 场景
我们可以给mysqld进程发送 SIGSEGV(11) 信号，即可模拟出coredump的场景，例如：
```
$ kill -s SIGSEGV `pidof mysqld`
```

这时查看GreatSQL错误日志文件，以及core文件，就会发现有coredump:

```
$ls -la 
...
-rw-------   1 mysql mysql 1081147392 Feb 20 22:36 core.mysqld-debug.2658134.1676903816
...

$ less error.log
...
14:36:56 UTC - mysqld got signal 11 ;
Most likely, you have hit a bug, but this error can also be caused by malfunctioning hardware.

Build ID: 1f4232b893100742b7c519df2fa714648c2d76d9
Server Version: 8.0.25-16-debug Source distribution

Thread pointer: 0x0
Attempting backtrace. You can use the following information to find out
where mysqld died. If you see no messages after this, something went
terribly wrong...
stack_bottom = 0 thread_stack 0x80000
/usr/local/GreatSQL-8.0.25-16-Linux-glibc2.28-x86_64/bin/mysqld-debug(my_print_stacktrace(unsigned char const*, unsigned long)+0x43) [0x4b04
d26]
/usr/local/GreatSQL-8.0.25-16-Linux-glibc2.28-x86_64/bin/mysqld-debug(handle_fatal_signal+0x2cb) [0x39a7d22]
/lib64/libpthread.so.0(+0x12c20) [0x7fc3e669ac20]
/lib64/libc.so.6(__poll+0x51) [0x7fc3e45c4a41]
/usr/local/GreatSQL-8.0.25-16-Linux-glibc2.28-x86_64/bin/mysqld-debug(Mysqld_socket_listener::listen_for_connection_event()+0x57) [0x3995195
]
/usr/local/GreatSQL-8.0.25-16-Linux-glibc2.28-x86_64/bin/mysqld-debug(Connection_acceptor<Mysqld_socket_listener>::connection_event_loop()+0
x30) [0x355a024]
/usr/local/GreatSQL-8.0.25-16-Linux-glibc2.28-x86_64/bin/mysqld-debug(mysqld_main(int, char**)+0x27d2) [0x354e4a6]
/usr/local/GreatSQL-8.0.25-16-Linux-glibc2.28-x86_64/bin/mysqld-debug(main+0x20) [0x32de906]
/lib64/libc.so.6(__libc_start_main+0xf3) [0x7fc3e44f6493]
/usr/local/GreatSQL-8.0.25-16-Linux-glibc2.28-x86_64/bin/mysqld-debug(_start+0x2e) [0x32de82e]
Please help us make Percona Server better by reporting any
bugs at https://bugs.percona.com/
...
```
信息采集：
+ 故障时刻的error log。
+ 故障产生的core文件。
+ 如果有general log的话，也采集起来（故障时刻往前约1小时或10万行日志）。
+ 导致core发生涉及到的表DDL以及相应的SQL语句，有必要的话，可能还要同时提供真实数据（或样例数据）。


## 初步分许
1. 分析error log
   首先需要在error log中，搜索backtrace关键字查找异常堆栈。
```
67912 Build ID: 0eaf4b944b1dbc99a26f9343f301e033bdedeb1d
67913 Server Version: 8.0.25-15-mysqlcluster5.0.7-GA MySQL Cluster, Release GA, Revision b43d2b2e462
67914
67915 Thread pointer: 0x7f914ec27000
67916 Attempting backtrace. You can use the following information to find out
67917 where mysqld died. If you see no messages after this, something went
67918 terribly wrong...
67919 stack_bottom = 7f91608d8c30 thread_stack 0x46000
67920 /mysql/svr/mysql/bin/mysqld(my_print_stacktrace(unsigned char const*, unsigned long)+0x3d) [0x200c33d]
67921 /mysql/svr/mysql/bin/mysqld(handle_fatal_signal+0x37b) [0x1169e4b]
67922 /lib64/libpthread.so.0(+0xf5d0) [0x7f91d95c55d0]
67923 /mysql/svr/mysql/lib/plugin/ha_mysql.so(mysql::ha_mysql::build_order_list_string(String&)+0x6c) [0x7f915951aa4c]
67924 /mysql/svr/mysql/lib/plugin/ha_mysql.so(mysql::ha_greatpart::rnd_init_low(bool)+0x1ac) [0x7f915959765c]
67925 /mysql/svr/mysql/bin/mysqld(handler::ha_rnd_init(bool)+0x22) [0xcc6102]
67926 /mysql/svr/mysql/bin/mysqld(TableScanIterator::Init()+0x4e) [0xeed66e]
67927 /mysql/svr/mysql/bin/mysqld(LimitOffsetIterator::Init()+0x16) [0x122afe6]
67928 /mysql/svr/mysql/bin/mysqld(Query_expression::ExecuteIteratorQuery(THD*)+0x281) [0x10a95a1]
67929 /mysql/svr/mysql/bin/mysqld(Query_expression::execute(THD*)+0x2f) [0x10a994f]
67930 /mysql/svr/mysql/bin/mysqld(Sql_cmd_dml::execute(THD*)+0x516) [0x1029376]
67931 /mysql/svr/mysql/bin/mysqld(mysql_execute_command(THD*, bool)+0x9e0) [0xfcce70]
67932 /mysql/svr/mysql/bin/mysqld(dispatch_sql_command(THD*, Parser_state*, bool)+0x4f1) [0xfd0961]
67933 /mysql/svr/mysql/bin/mysqld(dispatch_command(THD*, COM_DATA const*, enum_server_command)+0x1843) [0xfd26b3]
67934 /mysql/svr/mysql/bin/mysqld(do_command(THD*)+0x210) [0xfd3c70]
67935 /mysql/svr/mysql/bin/mysqld() [0x115ac90]
67936 /mysql/svr/mysql/bin/mysqld() [0x24cf38e]
67937 /lib64/libpthread.so.0(+0x7dd5) [0x7f91d95bddd5]
67938 /lib64/libc.so.6(clone+0x6d) [0x7f91d796dead]
67939
67940 Trying to get some variables.
67941 Some pointers may be invalid and cause the dump to abort.
67942 Query (7f913ef65028): select * from `grid-report`.`dp_bi_main_order_acccum` limit 0, 1000
67943 Connection ID (thread ID): 53
67944 Status: NOT_KILLED
```

得到如下信息:
+ Query (7f913ef65028): select * from grid-report.dp_bi_main_order_acccum limit 0, 1000语句导致的堆栈异常
+ 最后core在了build_order_list_string函数，且函数内的偏移是0x6c
+ Server Version: 8.0.25-15-mysqlcluster5.0.7-GA MySQL Cluster, Release GA, Revision b43d2b2e462, 这里是commit版本号，我们需要使用发布对应这个版本号的二进制。

2. 调试core 文件
```
# gdb /mysqld所在目录/mysqld /core文件所在目录/corefile
// 出现gdb命令行，敲入bt命令查看堆栈
(gdb) bt
#0  0x00007f6cd73619d1 in pthread_kill () from /lib64/libpthread.so.0
#1  0x0000000001169e7d in handle_fatal_signal (sig=11) at /builds/mysql-cluster/myrocks/sql/signal_handler.cc:194
#2  <signal handler called>
#3  operator() (__ptr=0x7f6c3636c71a, this=0x7f6c0e1b1f2c) at /opt/rh/devtoolset-10/root/usr/include/c++/10/bits/unique_ptr.h:79
#4  ~unique_ptr (this=0x7f6c0e1b1f2c, __in_chrg=<optimized out>) at /opt/rh/devtoolset-10/root/usr/include/c++/10/bits/unique_ptr.h:361
#5  ~Gdb_execute_plan (this=0x7f6c0e1b1f2c, __in_chrg=<optimized out>) at /builds/mysql-cluster/myrocks/storage/mysql/gdb_execute_plan.h:69
#6  operator() (this=0x7f6c3635c548, __ptr=0x7f6c0e1b1f2c) at /opt/rh/devtoolset-10/root/usr/include/c++/10/bits/unique_ptr.h:85
#7  operator() (__ptr=0x7f6c0e1b1f2c, this=0x7f6c3635c548) at /opt/rh/devtoolset-10/root/usr/include/c++/10/bits/unique_ptr.h:79
#8  ~unique_ptr (this=0x7f6c3635c548, __in_chrg=<optimized out>) at /opt/rh/devtoolset-10/root/usr/include/c++/10/bits/unique_ptr.h:361
#9  mysql::execute_direct_query(std::unique_ptr<mysql::Gdb_execute_node, std::default_delete<mysql::Gdb_execute_node> >&, mysql::Gdb_direct_exec_parameters*, unsigned long, bool) () at /builds/mysql-cluster/myrocks/storage/mysql/gdb_query.cc:1669
#10 0x00007f6c0e19b65c in mem_free (this=<optimized out>) at /builds/mysql-cluster/myrocks/include/sql_string.h:382
#11 ~String (this=<optimized out>, __in_chrg=<optimized out>) at /builds/mysql-cluster/myrocks/include/sql_string.h:235
#12 mysql::ha_greatpart::index_read_map_low(unsigned char const*, unsigned long, ha_rkey_function, unsigned int) ()
    at /builds/mysql-cluster/myrocks/storage/mysql/ha_greatpart.cc:881
#13 0x0000000000cc6102 in handler::ha_rnd_init (this=0x7f6c720fb028, scan=<optimized out>) at /builds/mysql-cluster/myrocks/sql/handler.cc:3141
#14 0x0000000000eed66e in TableScanIterator::Init (this=0x7f6b7f486c50) at /builds/mysql-cluster/myrocks/sql/row_iterator.h:213
#15 0x000000000122afe6 in LimitOffsetIterator::Init (this=0x7f6b7f486c88)
    at /opt/rh/devtoolset-10/root/usr/include/c++/10/bits/unique_ptr.h:421
#16 0x00000000010a95a1 in Query_expression::ExecuteIteratorQuery(THD*) ()
    at /opt/rh/devtoolset-10/root/usr/include/c++/10/bits/unique_ptr.h:421
#17 0x00000000010a994f in Query_expression::execute(THD*) () at /builds/mysql-cluster/myrocks/sql/sql_union.cc:1310
#18 0x0000000001029376 in Sql_cmd_dml::execute(THD*) () at /builds/mysql-cluster/myrocks/sql/sql_select.cc:575
#19 0x0000000000fcce70 in mysql_execute_command(THD*, bool) () at /builds/mysql-cluster/myrocks/sql/sql_parse.cc:4725
#20 0x0000000000fd0961 in dispatch_sql_command (thd=thd@entry=0x7f6beb676000, parser_state=parser_state@entry=0x7f6c3636daa0,
    update_userstat=update_userstat@entry=false) at /builds/mysql-cluster/myrocks/sql/sql_parse.cc:5321
#21 0x0000000000fd26b3 in dispatch_command(THD*, COM_DATA const*, enum_server_command) () at /builds/mysql-cluster/myrocks/sql/sql_parse.cc:1969
#22 0x0000000000fd3c70 in do_command (thd=0x7f6beb676000) at /builds/mysql-cluster/myrocks/sql/sql_parse.cc:1417
#23 0x000000000115ac90 in handle_connection (arg=arg@entry=0x7f6bed741340)
    at /builds/mysql-cluster/myrocks/sql/conn_handler/connection_handler_per_thread.cc:307
#24 0x00000000024cf38e in pfs_spawn_thread (arg=0x7f6bed4b63e0) at /builds/mysql-cluster/myrocks/storage/perfschema/pfs.cc:2899
#25 0x00007f6cd735cdd5 in start_thread () from /lib64/libpthread.so.0
#26 0x00007f6cd570cead in clone () from /lib64/libc.so.6
```
此信息需要截图。可以联系研发协助打印一些变量信息。

3. 信息收集
经过上面的初步分析，需要搜集如下文件提交给研发人员进一步分析。
+ error log
+ general log
+ core文件，如果研发之前已经搜集到了，可以不用传出来。
+ 导致core的query涉及到的表的建表语句以及数据（数据需要看研发是否需要）。
 
如果general log很大，需要DBA截取core之前1小时或者10万行的日志即可，可以通过grep、head、tail等命令截取相关日志。
+ 
+ 通过error日志确定core的时间点
+ 通过grep在general log中定位到具体时间点的行号 x
+ 通过head -n x general.log | tail -n 100000 > general.log.coredump
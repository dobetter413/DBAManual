# MySQL 逻辑备份

MySQL数据库自带的一个很好用的备份命令。是逻辑备份，导出 的是SQL语句。也就是把数据从MySQL库中以逻辑的SQL语句的形式直接输出或生成备份的文件的过程。

## 用法
```
Usage: mysqldump [OPTIONS] database [tables]
OR     mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]
OR     mysqldump [OPTIONS] --all-databases [OPTIONS]
```

## 关键参数
```
-A：备份所有库
-B：指定多个库，在备份文件中增加建库语句和use语句
-F：刷新binlog日志
--master-data：在备份文件中增加binlog日志文件名及对应的位置点,0\1\2,0为不添加该信息，2为注释形式添加
-d：只备份表结构
-t：只备份数据
--single-transaction：适合innodb事务数据库的备份
   InnoDB表在备份时，通常启用选项--single-transaction来保证备份的一致性，原理是设定本次会话的隔离级别为Repeatable read，来保证本次会话（也就是dump）时，不会看到其它会话已经提交了的数据。
--flush-privileges在导出mysql数据库之后，发出一条FLUSH PRIVILEGES 语句。为了正确恢复，该选项应该用于导出mysql数据库和依赖mysql数据库数据的任何时候。
-E--events：导出事件。
--routines, -R 导出存储过程以及自定义函数。
--triggers 导出触发器
--set-gtid-purged[=name] 
                     Add 'SET @@GLOBAL.GTID_PURGED' to the output. Possible
                      values for this option are ON, COMMEN TED, OFF and AUTO.
                      If ON is used and GTIDs are not enabled on the server, an
                      error is generated. If COMMENTED is used, 'SET
                      @@GLOBAL.GTID_PURGED' is added as a comment. If OFF is
                      used, this option does nothing. If AUTO is used and GTIDs
                      are enabled on the server, 'SET @@GLOBAL.GTID_PURGED' is
                      added to the output. If GTIDs are disabled, AUTO does
                      nothing. If no value is supplied then the default (AUTO)
                      value will be considered.

```
详细参数请参考附录


## 备份示例
1. 备份全库
```
mysqldump --uroot -proot -h127.0.0.1 -P3306  --all-databases --flush-privileges --single-transaction --master-data=2 --set-gtid-purged=OFF --flush-logs --triggers --routines --events --hex-blob > $BACKUP_DIR/full_dump_$BACKUP_TIMESTAMP.sql
```
2. 备份指定库
```
mysqldump --uroot -proot -h127.0.0.1 -P3306   --flush-privileges --single-transaction --master-data=2 --set-gtid-purged=OFF  --flush-logs --triggers --routines --events --hex-blob -B db1 db2 db3 > $BACKUP_DIR/full_dump_$BACKUP_TIMESTAMP.sql
```
3. 备份指定表
```
mysqldump --uroot -proot -h127.0.0.1 -P3306   --flush-privileges --single-transaction --master-data=2 --set-gtid-purged=OFF  --flush-logs --triggers --routines --events --hex-blob db1 table1 table2 > $BACKUP_DIR/full_dump_$BACKUP_TIMESTAMP.sql
```
4. 不备份数据
```
如果是选全部数据，则系统的mysql库还是带着数据，个人创建的是只有结构
mysqldump -uroot-proot -h127.0.0.1 -P3306 --all-databases --triggers --routines --events -d > all.sql
可以指定我们自己的库，那么备份出来的只有结构
mysqldump -uroot -proot -h127.0.0.1 -P3306 --triggers --routines --events  -d  db1 db2 > all.sql
mysqldump -uroot -proot -h127.0.0.1 -P3306 --triggers --routines --events    db1 table1 table2 > all.sql
```



## 备份原理

原理

```
1、关闭所有打开的表
2、调用FTWRL锁定所有的表，全局禁止更新
3、修改实例隔离级别
4、开启一致性快照读
5、获取gtid和binlog、pos信息
6、释放FTWRL锁
7、设置回滚点逐一备份数据表
8、备份完成
```

执行过程

```
1、FLUSH /*!40101 LOCAL */ TABLES    强制关闭所有正在使用的表
2、FLUSH TABLES WITH READ LOCK    关闭所有打开的表，并使用全局读锁锁定整个实例下的所有表
3、SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ   修改隔离级别为rr
4、START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */    开启一致性事物快照
5、SHOW VARIABLES LIKE 'gtid\_mode'    查询是否开启GTID
6、SELECT @@GLOBAL.GTID_EXECUTED    获取GTID信息（用于主从复制）
7、SHOW MASTER STATUS    获取binlog和pos信息（用于主从复制）
8、UNLOCK TABLES    释放表锁
9、SHOW DATABASES    获取所有库信息
10、app-plateform    进入app-plateform库
11、SHOW CREATE DATABASE IF NOT EXISTS `app-plateform`    获取创建库的预警
12、SAVEPOINT sp    设置事物回滚点   
13、show tables    获取库中所有表的信息
14、show table status like 'pub_alarm_history'    获取表的状态信息
15、show create table `pub_alarm_history`    获取创建表的语句
16、SELECT /*!40001 SQL_NO_CACHE */ * FROM `pub_alarm_history` 查询表中的数据
17、use `app-plateform`
18、SHOW TRIGGERS LIKE 'pub_alarm_history'    获取触发器
19、ROLLBACK TO SAVEPOINT sp    回滚到回滚点sp。ROLLBACK TO SAVEPOINT 语句还有一个作用，可以释放在设置保存点之后事务持有的MDL锁，这点便是mysqldump需要使用保存点的关键点。
重复12-19,直至所有表及数据备份完成
```

### DDL对备份的影响
官方文档:
>While a --single-transaction dump is in process, to ensure a valid dump file (correct table contents and binary log coordinates), no other connection should use the following statements: ALTER TABLE, CREATE TABLE, DROP TABLE, RENAME TABLE, TRUNCATE TABLE. A consistent read is not isolated from those statements, so use of them on a table to be dumped can cause the SELECT that is performed by mysqldump to retrieve the table contents to obtain incorrect contents or fail.

翻译:
>在 --single-transaction转储过程中，为确保转储文件有效（正确的表内容和二进制日志坐标），其他任何连接都不应使用以下语句： ALTER TABLE, CREATE TABLE, DROP TABLE, RENAME TABLE, TRUNCATE TABLE。一致性读取并不与这些语句隔离，因此在要转储的表上使用它们可能会导致 mysqldumpSELECT执行的 检索表内容的操作获取不正确的内容或失败。

*结论，备份期间的DDL 可能会导致备份出现不一致的状态，所以备份期间要避免出现DDL 操作。*

实测:
case1: 在(2-8) 过程之间锁表，不允许DDL、DML
case2: (8-14) 过程之间如果发生DML、DDL，事务号发生变化，DML 没有影响，DDL
DROP table\database 在RR 级别下还可以通过show databases、tables为删除前的状态，但show create database\table 或者select * from xxx 会发生报错，`ERROR 1049 (42000): Unknown database 'xxx' 或者 ERROR 1146 (42S02): Table 'xxx.xxx' doesn't exist`
经测试发现，对表字段增删、主键添加删除等备份会报错，普通索引增删备份不报错




## 注意事项
1. 备份mysql 库后需要刷新权限,添加备份选项 `--flush-privileges`

## 踩坑日记

### 问题现象
通过如下命令对数据库进行备份
```
/greatdb/bin/greatdbdump -u$bak_user -p$bak_password  -h$bak_host -P$bak_port --set-gtid-purged=OFF -E -R --triggers --master-data=2 --single-transaction -A  > /backup/20230228.sql
```

通过如下方式`source /backup/20230228.sql`在导入一个MySQL 新实例时发现报错如下。

```
ERROR 1449 (HY000): The user specified as a definer ('yy'@'%') does not exist
ERROR 1146 (42S02): Table 'iuap_apdoc_basedoc.v_cmcc_jsxjh' doesn't exist
```
### 问题定位
1. 通过以上报错，可以推断是在创建时视图或者存储过程和函数、存储过程这种数据库对象时产生的报错。
2. 通过查看源库和恢复出来的新库的系统表，对比以上数据库对象是否存在缺失，定位到缺少两个视图。
   
```
select TABLE_SCHEMA,TABLE_NAME from information_schema.views where table_schema !='sys';

select ROUTINE_SCHEMA,ROUTINE_NAME,ROUTINE_TYPE from information_Schema.ROUTINES where ROUTINE_SCHEMA!='sys';

select EVENT_SCHEMA,EVENT_NAME from information_schema.EVENTS;
```

3. 通过一下两种方式查看视图创建语句
+ 在源库查看创建视图语句
  `show create view  xxxx;`
+ 在备份文件中过滤视图创建语句
  
  ```
    echo  'USE `iuap_apdoc_basedoc`;'  > ./addview.sql 
    echo  "flush privileges;"  >>./addview.sql 
    grep  'Final view structure for view `v_cmcc_jsxjh`'  -B 1 -A 16 ./20230228.sql  >> ./addview.sql 
    grep  'Final view structure for view `v_cmcc_zhjh`'   -B 1 -A 16 ./20230228.sql  >> ./addview.sql 
  ```

4. 然后手工执行创建语句，复现了  `ERROR 1449 (HY000): The user specified as a definer ('yy'@'%') does not exist` 报错
5. 确认用户是否存在   `select user,host from mysql.user;` 用户存在

### 问题处理
1. 刷新内存中的acl信息 `flush privileges;`
2. 重新执行创建view，执行成功
### 问题分析
1. 全备包含了 mysql.user 表，此表数据通过insert 方式导入，涉及的用户未及时加载到mysql 内存 acl中
2. 通过DML 方式跟新 mysql.user 表不会立即更新内存中的acl，需要执行 `flush privileges;` 手工更新
3. 通过grant、create user 等方式更改用户会及时刷新到内存和磁盘，不存在此问题
4. 后面查看了 mysqldump 的参数选项，发现了`--flush-privileges` 选项，在备份mysql 库后添加 `flush privileges;` 操作，用来规避此问题。


# 附录
## 参数解析
```
-A --all-databases：导出全部数据库
-Y --all-tablespaces：导出全部表空间
-y --no-tablespaces：不导出任何表空间信息
--add-drop-database每个数据库创建之前添加drop数据库语句。
--add-drop-table每个数据表创建之前添加drop数据表语句。(默认为打开状态，使用--skip-add-drop-table取消选项)
--add-locks在每个表导出之前增加LOCK TABLES并且之后UNLOCK TABLE。(默认为打开状态，使用--skip-add-locks取消选项)
--comments附加注释信息。默认为打开，可以用--skip-comments取消
--compact导出更少的输出信息(用于调试)。去掉注释和头尾等结构。可以使用选项：--skip-add-drop-table --skip-add-locks --skip-comments --skip-disable-keys
-c --complete-insert：使用完整的insert语句(包含列名称)。这么做能提高插入效率，但是可能会受到max_allowed_packet参数的影响而导致插入失败。
-C --compress：在客户端和服务器之间启用压缩传递所有信息
-B--databases：导出几个数据库。参数后面所有名字参量都被看作数据库名。
--debug输出debug信息，用于调试。默认值为：d:t:o,/tmp/
--debug-info输出调试信息并退出
--default-character-set设置默认字符集，默认值为utf8
--delayed-insert采用延时插入方式（INSERT DELAYED）导出数据
-E--events：导出事件。
--master-data：在备份文件中写入备份时的binlog文件，在恢复进，增量数据从这个文件之后的日志开始恢复。值为1时，binlog文件名和位置没有注释，为2时，则在备份文件中将binlog的文件名和位置进行注释
--flush-logs开始导出之前刷新日志。请注意：假如一次导出多个数据库(使用选项--databases或者--all-databases)，将会逐个数据库刷新日志。除使用--lock-all-tables或者--master-data外。在这种情况下，日志将会被刷新一次，相应的所以表同时被锁定。因此，如果打算同时导出和刷新日志应该使用--lock-all-tables 或者--master-data 和--flush-logs。
--flush-privileges在导出mysql数据库之后，发出一条FLUSH PRIVILEGES 语句。为了正确恢复，该选项应该用于导出mysql数据库和依赖mysql数据库数据的任何时候。
--force在导出过程中忽略出现的SQL错误。
-h --host：需要导出的主机信息
--ignore-table不导出指定表。指定忽略多个表时，需要重复多次，每次一个表。每个表必须同时指定数据库和表名。例如：--ignore-table=database.table1 --ignore-table=database.table2 ……
-x --lock-all-tables：提交请求锁定所有数据库中的所有表，以保证数据的一致性。这是一个全局读锁，并且自动关闭--single-transaction 和--lock-tables 选项。
-l --lock-tables：开始导出前，锁定所有表。用READ LOCAL锁定表以允许MyISAM表并行插入。对于支持事务的表例如InnoDB和BDB，--single-transaction是一个更好的选择，因为它根本不需要锁定表。请注意当导出多个数据库时，--lock-tables分别为每个数据库锁定表。因此，该选项不能保证导出文件中的表在数据库之间的逻辑一致性。不同数据库表的导出状态可以完全不同。
--single-transaction：适合innodb事务数据库的备份。保证备份的一致性，原理是设定本次会话的隔离级别为Repeatable read，来保证本次会话（也就是dump）时，不会看到其它会话已经提交了的数据。
-F：刷新binlog，如果binlog打开了，-F参数会在备份时自动刷新binlog进行切换。
-n --no-create-db：只导出数据，而不添加CREATE DATABASE 语句。
-t --no-create-info：只导出数据，而不添加CREATE TABLE 语句。
-d --no-data：不导出任何数据，只导出数据库表结构。
-p --password：连接数据库密码
-P --port：连接数据库端口号
-u --user：指定连接的用户名。
--routines, -R 导出存储过程以及自定义函数。
--triggers 导出触发器
--max-allowed-packet=#  接收和发送的数据包最大值 The maximum packet length to send to or receive from
--opt               Same as --add-drop-table, --add-locks, --create-options,
                      --quick, --extended-insert, --lock-tables, --set-charset,
                      and --disable-keys. Enabled by default, disable with
                      --skip-opt.

```

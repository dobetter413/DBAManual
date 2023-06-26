# my2sql 工具
>此文档使用场景为使用my2sql 解析binlog，分析原始语句或生成回滚语句


## 重要参数
```
-mode  repl 伪装slave 解析binlog，file 离线解析binlog 文件，默认为repl 模式
-local-binlog-file  当指定-mode=file时，需使用该参数执行binlog 文件的路径
-databases、-tables 库表过滤条件
-sql  要解析的sql 类型，可选参数 insert|update|delete,默认全部解析
-doNotAddPrifixDb 生成不带库名的sql，默认生成的为 db.tb
-file-per-table 为每个表生成一个sql文件
-full-columns 生成的sql是否带全列信息
-ignorePrimaryKeyForInsert 生成的insert 语句是否去掉主键
-output-dir 将生成的结果存放到指定目录
-threads 线程数，默认8个
-work-type 2sql:生成原始sql，rollback:生成回滚sql,stats：只统计DML、事务信息
```
参考文档  https://github.com/liuhr/my2sql/blob/master/README.md

## 使用案例
1. 解析出标准SQL
+ 根据时间点解析出标准SQL
  + 伪装成从库解析binlog
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode repl -work-type 2sql  -start-file mysql-bin.011259  -start-datetime "2020-07-16 10:20:00" -stop-datetime "2020-07-16 11:00:00" -output-dir ./tmpdir
```
  + 直接读取binlog文件解析
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode file -local-binlog-file ./mysql-bin.011259  -work-type 2sql  -start-file mysql-bin.011259  -start-datetime "2020-07-16 10:20:00" -stop-datetime "2020-07-16 11:00:00" -output-dir ./tmpdir
```

+ 根据pos点解析出标准SQL
  + 伪装成从库解析binlog
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode repl  -work-type 2sql  -start-file mysql-bin.011259  -start-pos 4 -stop-file mysql-bin.011259 -stop-pos 583918266  -output-dir ./tmpdir
```
  + 直接读取binlog文件解析
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306  -mode file -local-binlog-file ./mysql-bin.011259  -work-type 2sql  -start-file mysql-bin.011259  -start-pos 4 -stop-file mysql-bin.011259 -stop-pos 583918266  -output-dir ./tmpdir
```


2. 解析出回滚SQL
+ 根据时间点解析出回滚SQL
  + 伪装成从库解析binlog
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode repl -work-type rollback  -start-file mysql-bin.011259  -start-datetime "2020-07-16 10:20:00" -stop-datetime "2020-07-16 11:00:00" -output-dir ./tmpdir
```
  + 直接读取binlog文件解析
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306  -mode file -local-binlog-file ./mysql-bin.011259 -work-type rollback  -start-file mysql-bin.011259  -start-datetime "2020-07-16 10:20:00" -stop-datetime "2020-07-16 11:00:00" -output-dir ./tmpdir
```

+ 根据pos点解析出回滚SQL
  + 伪装成从库解析binlog
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode repl -work-type rollback  -start-file mysql-bin.011259  -start-pos 4 -stop-file mysql-bin.011259 -stop-pos 583918266  -output-dir ./tmpdir
```
  + 直接读取binlog文件解析
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306   -mode file -local-binlog-file ./mysql-bin.011259  -work-type rollback  -start-file mysql-bin.011259  -start-pos 4 -stop-file mysql-bin.011259 -stop-pos 583918266  -output-dir ./tmpdir
```

3. 统计DML以及大事务
+ 统计时间范围各个表的DML操作数量，统计一个事务大于500条、时间大于300秒的事务
  + 伪装成从库解析binlog
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306  -mode repl -work-type stats  -start-file mysql-bin.011259  -start-datetime "2020-07-16 10:20:00" -stop-datetime "2020-07-16 11:00:00"  -big-trx-row-limit 500 -long-trx-seconds 300   -output-dir ./tmpdir
```
  + 直接读取binlog文件解析
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode file -local-binlog-file ./mysql-bin.011259   -work-type stats  -start-file mysql-bin.011259  -start-datetime "2020-07-16 10:20:00" -stop-datetime "2020-07-16 11:00:00"  -big-trx-row-limit 500 -long-trx-seconds 300   -output-dir ./tmpdir
```
+ 统计一段pos点范围各个表的DML操作数量，统计一个事务大于500条、时间大于300秒的事务
  + 伪装成从库解析binlog
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306  -mode repl -work-type stats  -start-file mysql-bin.011259  -start-pos 4 -stop-file mysql-bin.011259 -stop-pos 583918266  -big-trx-row-limit 500 -long-trx-seconds 300   -output-dir ./tmpdir
```
  + 直接读取binlog文件解析
```
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode file -local-binlog-file ./mysql-bin.011259  -work-type stats  -start-file mysql-bin.011259  -start-pos 4 -stop-file mysql-bin.011259 -stop-pos 583918266  -big-trx-row-limit 500 -long-trx-seconds 300   -output-dir ./tmpdir
```
从某一个pos点解析出标准SQL，并且持续打印到屏幕
#伪装成从库解析binlog
./my2sql  -user root -password xxxx -host 127.0.0.1   -port 3306 -mode repl  -work-type 2sql  -start-file mysql-bin.011259  -start-pos 4   -output-toScreen 



## 示例
```
解析出回滚SQL
./my2sql  -user root -password '!QG2e@TdB'  -port 16321 -host 192.168.96.7 -databases app_grid  -tables plat_wf_proc_task -work-type 2sql   -start-file greatdb-bin.003399 -start-datetime "2022-09-02 21:00:00" --stop-datetime "2022-09-02 21:30:00" -output-dir ./tmp/

./my2sql  -user root -password '!QG2e@TdB'  -port 16321 -host 192.168.96.7 -databases app_grid  -tables plat_wf_proc_task -work-type rollback   -start-file greatdb-bin.003399 -start-datetime "2022-09-02 21:00:00" --stop-datetime "2022-09-02 21:30:00"  -output-dir ./tmp/
```

## 使用限制
+ 使用回滚/闪回功能时，binlog格式必须为row,且binlog_row_image=full， DML统计以及大事务分析不受影响
+ 只能回滚DML， 不能回滚DDL
+ 支持指定-tl时区来解释binlog中time/datetime字段的内容。开始时间-start-datetime与结束时间-stop-datetime也会使用此指定的时区， 但注意此开始与结束时间针对的是binlog event header中保存的unix timestamp。结果中的额外的datetime时间信息都是binlog event header中的unix timestamp
+ 此工具是伪装成从库拉取binlog，需要连接数据库的用户有SELECT, REPLICATION SLAVE, REPLICATION CLIENT权限
+ MySQL8.0版本需要在配置文件中加入default_authentication_plugin =mysql_native_password，用户密码认证必须是mysql_native_password才能解析


## 编译安装
```
git clone https://github.com/liuhr/my2sql.git
cd my2sql/
go build .
```

>工具存放至百度网盘 项目安装介质中

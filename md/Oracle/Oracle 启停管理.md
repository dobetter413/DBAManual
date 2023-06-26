# Oracle 启停管理

## 1.Oracle 数据库启动
### 1).启动阶段

> shutdown -> nomount -> mount ->open
![](images/2022-11-28-17-42-59.png)


### 2).启动命令
```
sqlplus / as sysdba
startup
startup force  #强制启动
```


### 3).启动监听
```
su - oracle
lsnrctl start
lsnrctl status
```

## 2.Oracle 数据库停止

### 1).关闭模式
> 中止(abort)、立即(immediate)、正常(normal)、事务 (transactional)

| |终止|立即|正常|事务性|
|--|--|--|--|--|
|允许新连接|no|no|no|no|
|等待当前会话结束|no|no|yes|no|
|等待当前事务结束|no|no|yes|yes|
|强制检查点和关闭文件|no|yes|yes|yes|


### 2).关闭指令
```
shutdown abort          #终止
shutdown immediate      #立即
shutdown normal         #正常
shutdown transactional  #事务性
```

### 3).关闭监听
```
su - oracle
lsnrctl stop
lsnrctl status
```

> 只关闭数据库不关闭监听时 1521 端口会存在，但是无法登录。

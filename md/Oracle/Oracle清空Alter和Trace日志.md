## 日志保存位置
Trace日志保存位置：

/oracle/app/oracle/diag/tnslsnr/xxx/listener/trace
Alert日志保存位置：

/oracle/app/oracle/diag/tnslsnr/xxx/listener/alert
可以使用如下命令查询日志记录目录：

`sqlplus / as systemdba`
## 查询日志路径
`SQL>select * from v$diag_info;`

## 清空trace日志
### 停止trace日志写入
1. 切换到oracle用户
`su - oracle`
 
2. 停止监听
`lsnrctl stop`
 
3. 进入监听日志位置
`/oracle/app/oracle/diag/tnslsnr/xxx/listener/trace`
 
4. 删除日志
`rm -rf *.log`
 
5. 启动监听
`lsnrctl start`

## 关闭监听日志
1. 切换到oracle用户
`su - oracle`
 
2. 打开监听配置
`lsnrctl `
3. 关闭监听日志
`set log_status off`
4. 保存配置
`save_config`
5. 清空alert日志
进入alert目录
`cd /oracle/app/oracle/diag/tnslsnr/xxx/listener/alert`
 
6. 备份
`mv * /data/back`


## 问题处理
ORA-03113: end-of-file on communication channel
https://www.jianshu.com/p/97b86d95894b
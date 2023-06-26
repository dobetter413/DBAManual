# DataX 数据迁移工具

>DataX 是阿里巴巴集团内被广泛使用的离线数据同步工具/平台，实现包括 MySQL、SQL Server、Oracle、PostgreSQL、HDFS、Hive、HBase、OTS、ODPS 等各种异构数据源之间高效的数据同步功能。

### 依赖
+ Linux
+ JDK(1.8以上，推荐1.8)
+ Python(2或3都可以)
+ Apache Maven 3.x (Compile DataX)

### 部署
1. 配置java 环境和python 环境
2. 下载DataX 工具包
   下载地址 https://datax-opensource.oss-cn-hangzhou.aliyuncs.com/202210/datax.tar.gz
   下载后解压至本地某个目录，进入bin目录，即可运行同步作业:
   ```
    $ cd  {YOUR_DATAX_HOME}/bin
    $ python datax.py {YOUR_JOB.json}
   ```

3. 生成任务配置示例
   python ./bin/datax.py -r oraclereader -w mysqlwriter  > ./job/o2m.json

4. 根据示例修改 o2m.json 文件

5. 执行任务
   python bin/datax.py  --jvm="-Xms2G -Xmx8G" ./job/o2m.json
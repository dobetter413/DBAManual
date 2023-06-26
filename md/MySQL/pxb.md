

本地备份
xtrabackup  \
--defaults-file=/edb/mysqldata/16315fff-2c7d-477b-aaef-dce751338829/my16315.conf \
-uadmin -p'!QAZ2wsx' --backup -P16315 --host=127.0.0.1 --target-dir=/pxbtest/backup

恢复
prepare 
xtrabackup --defaults-file=/edb/mysqldata/16315fff-2c7d-477b-aaef-dce751338829/my16315.conf  --prepare  \
--target-dir=/pxbtest/back
(prepare 可以不加配置文件)

copy back 
xtrabackup --defaults-file=/edb/mysqldata/16315fff-2c7d-477b-aaef-dce751338829/my16315.conf -uadmin -p'!QAZ2wsx' \
-P3307 --host=127.0.0.1 --copy-back  --target-dir=/soft/backup/
(copy back 可以不加配置文件和登录信息)

本地流式备份
xtrabackup  \
--defaults-file=/edb/mysqldata/16315fff-2c7d-477b-aaef-dce751338829/my16315.conf \
-uadmin -p'!QAZ2wsx' --backup -P16315 --host=127.0.0.1 \
--stream=xbstream  |cat >/pxbtest/xbtest.stream

xbstream -x < ./backup.xbstram -C ./back
解压后同普通恢复流程，prepare、copy back 


远程流式备份
最好开始screen 会话后备份，防止备份期间会话断开
需配置ssh 免密

方式一：
xtrabackup  \
--defaults-file=/edb/mysqldata/16315fff-2c7d-477b-aaef-dce751338829/my16315.conf \
-uadmin -p'!QAZ2wsx' --backup -P16315 --host=127.0.0.1 \
--stream=xbstream  --target-dir=/pxbtest/ |ssh root@172.16.50.88 "xbstream -x -C /pxbtest88/"

方式二：
xtrabackup  \
--defaults-file=/edb/mysqldata/16315fff-2c7d-477b-aaef-dce751338829/my16315.conf \
-uadmin -p'!QAZ2wsx' --backup -P16315 --host=127.0.0.1 \
--stream=xbstream  --target-dir=/pxbtest/ |ssh root@172.16.50.88 "cat - >/pxbtest88/pxb.stream"

xbstream -x < ./backup.xbstram -C ./back
解压后同普通恢复流程，prepare、copy back 





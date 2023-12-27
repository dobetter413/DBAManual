

## 系统配置优化
### CPU 设置性能模式
cpupower设置performance
```
cpupower frequency-set -g "performance"
```
检查cpu模式
```
cpupower frequency-info
```

performance模式检查
```
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```
预期输出：performance

注：如果该步报文件不存在，则需要安装 kernel-tools

CentOS 安装 kernel-tools

```
yum install kernel-tools -y
```


Ubuntu 安装 CPU 模式无图形化切换器
```
apt install cpufrequtils
```


### 内存和SWAP
swappiness值越大，越积极使用swap分区，值越小，越积极使用内存。
设置SWAP
```
vi /etc/sysctl.conf
echo "vm.swappiness = 1" >> /etc/sysctl.conf
sysctl -p
```
或者
```
sysctl vm.swappiness=1
```

查看swap 配置
```
cat /proc/sys/vm/swappiness

或

sysctl -q vm.swappiness

必要情况下，可关闭交换分区

swapoff -a
```

SWAP 优化
特殊情况下，如内存实在不够用，为避免SWAP落到性能差的磁盘，可用以下方法指定SWAP路径
```
cd /$disk_name
dd if=/dev/zero of=/$disk_name/swapfile bs=1024 count=1048576
mkswap swapfile
swapon swapfile
swapon -s
```

### 磁盘相关
#### 磁盘选型
```
选择SSD或者PCIe SSD设备，至少获得数百倍甚至万倍的IOPS提升。

机械盘分为SATA和SAS接口，转速越高越好，如15KRPM>7.2KRPM

SATA(3.0版本可达带宽6Gb/s，速度600MB/s)，适合大容量、非关键业务
SAS 适合高性能、高可靠业务

HHD混合硬盘(磁性碟片和闪存集成) > HDD机械硬盘
```

####  RAID 策略
```
从性能和安全综合考虑，RAID策略依次推荐RAID10、0、5。极限场景下可使用RAID0

机械盘：设置阵列写策略为write back，先写阵列卡缓存，甚至force WB

SSD盘：设置阵列写策略为write through，直接写磁盘性能更高，关闭RAID预读，使用direct io

购置阵列卡同时如果配备CACHE及BBU模块，可明显提升IOPS（主要是指机械盘，SSD或PCIe SSD除外。同时需要定期检查CACHE及BBU模块的健康状况，确保意外时不至于丢失数据）
```

####  IO 调度和4K 对齐
一般数据库环境选择deadline，如果涉及写场景性能测试且SSD/PCIE 或者 虚机，可以改为noop
```
echo noop > /sys/block/sda/queue/scheduler
```
配置IO调度，deadline或者noop更适用于MySQL数据库场景。命令中的sda为数据盘名称，根据实际磁盘名称进行修改。NVME盘不支持此操作。

注：PCIE接口的 SSD最优IO调度算法就是不用操作系统管，所以选择noop可能性能更好。而且所有跑在虚拟主机上的任何数据库实例也是选noop最好！避免IO走两层调度算法耽误时间

4K对齐：做磁盘分区时需要保证4K对齐，检查方法如下
```
fdisk -l
```
查看每个分区的start是否可被8整除，如是说明对齐；如不是，需重做分区
注：4K对齐对磁盘性能至关重要，一般影响30%左右(SSD)，机械盘影响较小

#### 文件系统和挂载

建议使用xfs高性能文件系统，在初始部署前，应确认数据库服务器使用xfs文件系统

MySQL数据分区独立，避免受其他分区的影响
Mount参数：
XFS: defaults,noatime,nodiratime,nobarrier,allocsize=256m,logbufs=8,attr2,logbsize=256k
mount -o remount /

说明：

1.SSD+EXT4方式的discard选项启用后，文件系统上的文件一旦被删除，会立即通知 SSD 进行 Trim 操作，这就是online discard (立即删除)。在进行删除大量小文件的操作时可能会造成不小的性能下降

参考：https://patrick-nagel.net/blog/archives/337

1.Linux会给文件记录了三个时间，
```
change time
modify time
access time
```
一般来说，文件都是读多写少，而且我们也很少关心某一个文件最近什么时间被访问了。所以，我们建议采用noatime选项，文件系统在程序访问对应的文件或者文件夹时，不会更新对应的access time。这样文件系统不记录access time，避免浪费资源。
另外，现在的很多文件系统会在数据提交时强制底层设备刷新cache，避免数据丢失，称之为write barriers。数据库服务器底层存储设备要么采用RAID卡，RAID卡本身的电池可以掉电保护；要么采用Flash卡，它也有自我保护机制，保证数据不会丢失。所以我们可以安全的使用nobarrier挂载文件系统。xfs文件系统可以指定nobarrier选项。


### 操作系统调优

#### numa配置
检查numa是否开启
```
grep -i numa /var/log/dmesg

输出结果为 No NUMA configuration found

说明numa为disable
```
或者
```
cat /proc/sys/kernel/numa_balancing

值为0说明未开启
```

关闭自动化NUMA平衡，减少MySQL线程在NUMA节点之间的切换
```
sysctl -w kernel.numa_balancing=0
```

将MySQL的进程绑定到指定CPU核上，同时内存在指定node交叉分配内存（例如绑定到core0-core92，内存分配node0-node3的内存）
```
numactl -C 0-92 -i 0-3 ../mysql/bin/mysqld --defaults-file=./my.cnf --user=mysql &
```

如果NUMA架构下实例内存分配不均匀
解决方案：临时修改numa内存分配策略为 interleave=all，即
```
numactl --interleave=all ../mysql/bin/mysqld --defaults-file=./my.cnf --user=mysql &
```

检查内存的node分配情况
```
numactl --hardware
或
numactl -H
```

注意：

1.鲲鹏服务器建议开启numa，绑定情况视numa具体情况与DBA讨论执行

在数据库实例启动之前drop cache，便于重新分配内存
```
sync && echo 3 >/proc/sys/vm/drop_caches
```

或者
```
sysctl -q -w vm.drop_caches=3
```

2.多路情况下（numa node > 2）如果buffer pool size小于单个numa node的内存，可将实例绑定到某个numa node；
```
numactl --interleave=0 ../mysqld_safe --defaults-file=my.cnf --innodb-numa-interleave=ON --user=mysql &
```

反之，如果单实例或buffer pool size大于单个numa node的内存，可由所有numa node来分配内存
```
numactl --interleave=all ../mysqld_safe --defaults-file=my.cnf --innodb-numa-interleave=ON --user=mysql &
```

3.启动后查看内存分配
```
pidof mysqld
numastat -p $mysqld_pid
```

#### 关闭irqbalance
关闭irqbalance，通过手动绑定网卡中断的方法优化性能
```
systemctl stop irqbalance.service
systemctl disable irqbalance.service
systemctl status irqbalance.service
```

#### 关闭sched_autogroup
关闭内核桌面交互优化，在高并发场景，桌面交互优化会降低主机性能
```
sysctl -w kernel.sched_autogroup_enabled=0
```

#### 关闭唤醒抢占
关闭唤醒抢占，减少进程被频繁睡眠/唤醒导致的性能问题
```
echo NO_WAKEUP_PREEMPTION > /sys/kernel/debug/sched_features
```

### BIOS 设置
1. 选择Performance Per Watt Optimized(DAPC)模式，发挥CPU最大性能，跑DB这种通常需要高运算量的服务就不要考虑节电了
2. 关闭C1E和C States等选项，目的也是为了提升CPU效率
3. Memory Frequency（内存频率）选择Maximum Performance（最佳性能）
4. 内存设置菜单中，启用Node Interleaving，避免NUMA问题
5. 关闭SMMU(非虚拟化场景使用)
    重启服务器过程中，进入BIOS -- MISC Config -- Support Smmu设置为Disable
6. 关闭预取
    重启服务器过程中，进入BIOS -- MISC Config -- CPU Prefetching Configuration设置为Disabled
7. 开启bios的turbo boost (Intel)
8. 关闭RAID卡的预读
    一般配置项为 Read Policy，预读选项一般为Always Read Ahead\Read Ahead\Ahead字样，不选择上述字样即可
9. RAID写入策略
    机械盘：设置阵列写策略为write back，先写阵列卡缓存，甚至force WB
    SSD盘：设置阵列写策略为write through，直接写磁盘性能更高，关闭RAID预读，使用direct io
    条带大小设置为64KB(该阵列卡条带最低为64KB，可设置16KB或32KB，提升并发写入性能) 
    经测试单机mysql，将stripe size从256k调整到64k，依据并发度的不同，性能提升有10%-25%


### 内核参数
#### 内存和SWAP
```
sysctl -w vm.dirty_ratio=10            #内存里的脏数据填充的百分比的绝对最大量
sysctl -w vm.dirty_background_ratio=10 #内存可填充脏数据的百分比
sysctl -w vm.swappiness=1              #换出运行时内存的相对权重
sysctl -w kernel.shmmax=68719476736    #单个共享内存段的最大值，过低会造成多个共享段
sysctl -w kernel.shmall=16777216       #可以使用的共享内存的总页数
```

#### 网络参数
```
sysctl -w net.ipv4.tcp_max_syn_backlog=819200              #指定所能接受SYN同步包的最大客户端数量。默认值是2048
sysctl -w net.core.netdev_max_backlog=400000               #系统中最多有多少TCP套接字不被关联到任何一个用户文件句柄
sysctl -w net.core.somaxconn=4096                          #服务端所能accept即处理数据的最大客户端数量，即完成连接上限。默认值是128
echo 16777216 > /proc/sys/net/core/rmem_max                #接收套接字缓冲区大小的最大值。默认值是229376，建议修改成16777216
echo 16777216 > /proc/sys/net/core/wmem_max                #发送套接字缓冲区大小的最大值（以字节为单位）。默认值是229376，建议修改成16777216
echo "4096 87380 16777216" > /proc/sys/net/ipv4/tcp_rmem   #配置读缓冲的大小，三个值，第一个是这个读缓冲的最小值，第三个是最大值，中间的是默认值。默认值是"4096 87380 6291456"，建议修改成"4096 87380 16777216"
echo "4096 65536 16777216" > /proc/sys/net/ipv4/tcp_wmem   #配置写缓冲的大小，三个值，第一个是这个写缓冲的最小值，第三个是最大值，中间的是默认值。默认值是"4096 16384 4194304"，建议修改成"4096 65536 16777216"
echo 360000 > /proc/sys/net/ipv4/tcp_max_tw_buckets        #表示系统同时保持TIME_WAIT套接字的最大数量，默认值是2048，建议修改成360000
```

#### TIMT_WAIT 优化
如遇到tcp短连接大量TIME_WAIT问题，可设置如下参数解决，需要根据实际情况谨慎设置
```
sysctl -w net.ipv4.tcp_tw_reuse=1        #当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭
sysctl -w net.ipv4.tcp_tw_recycle=1      #表示允许重用TIME_WAIT状态的套接字用于新的TCP连接,默认为0，表示关闭(NAT 网络环境有坑，偶发连接不上（LVS、F5）)
sysctl -w net.ipv4.tcp_syncookies = 1    #开启SYN Cookies
sysctl -w net.ipv4.tcp_fin_timeout = 30  #修改系統默认的 TIMEOUT 时间
```

#### 磁盘参数
```
echo 2048 > /sys/block/sda/queue/nr_requests   #提升磁盘吞吐量，可以调整到更大(默认128，改大可能会占用多些内存)。命令中的sda为数据盘名称，根据实际磁盘名称进行修改。
echo 16 > /sys/block/sda/queue/read_ahead_kb   #减少预读，默认128
```

### 启用jemalloc 
依赖jemalloc库，先安装
```
yum -y install jemalloc jemalloc-devel
```
也可以把自行安装的lib库so文件路径加到系统配置文件中
```
cat /etc/ld.so.conf

输出
/usr/local/lib64/
执行下面的操作加载libjemalloc库，确认是否已存在

ldconfig

ldconfig -p | grep libjemalloc

输出

libjemalloc.so.1 (libc6,x86-64) => /usr/local/lib64/libjemalloc.so.1
libjemalloc.so (libc6,x86-64) => /usr/local/lib64/libjemalloc.so
```

数据库实例启动之前执行
```
export LD_PRELOAD=/usr/local/lib64/libjemalloc.so
```

对比：
glibc：数据库默认，基于ptmalloc2，性能弱于tcmalloc和jemalloc
tcmalloc：google开源，小对象内存malloc和free时间上，ptmalloc大概是tcmalloc的6倍，以及并发场景
jemalloc：facebook，降低内存碎片化，优势同tcmalloc(并发、多核场景)
参考资料：
github使用tcmalloc后，mysql性能提升30% https://github.com/blog/1422-tcmalloc-and-mysql
内存优化总结:ptmalloc、tcmalloc和jemalloc http://www.cnhalo.net/2016/06/13/memory-optimize/


### 网卡
绑定网卡中断
根据网卡所属CPU将其进行分配，从而优化系统网络性能。一般将压测机器网卡中断绑定在对应numa node上

先使用如下命令确认网卡中断所在numa node的cpulist
```
eth=<网卡名称>                                     #网卡名称用命令，使用 ip a 查看
node_id=`cat /sys/class/net/$eth/device/numa_node`
cat /sys/devices/system/node/node$node_id/cpulist
```

上面的命令将会输出一个cpulist，例如48-71，然后使用 smartirq.sh脚本将网卡中断绑定到这些CPU上，命令如下
```
./smartirq.sh $eth 48 71                          #48和71是上面的命令输出的cpulist范围
```
当cat /sys/devices/system/node/node0/cpulist
输出为：0,4,8,12,16,20,24,28,32,36,40,44,48格式时，使用如下方式绑定网卡中断

首先需要获取几个参数，如下

查看numa节点数和每个numa节点逻辑cpu数量
```
numactl --hardware
```
查看网卡挂在哪个numa节点上
```
cat /sys/class/net/$eth/device/numa_node

或

numactl --prefer netdev:$eth --show
```
查看CPU超线程
```
lscpu | grep 'Thread(s) per core:'
```

然后执行./bindirq.sh $eth cpus_per_numa numas numa_node thread_per_core绑定

示例
```
./bindirq.sh $eth 44 4 0 2
```

smartirq.sh脚本
```
#!/bin/bash
if [ $# != 3 ] ; then
    echo "USAGE: $0 eth_name start_cpu_core end_cpu_core"
    echo " e.g.: $0 enp130s0f0 0 63"
    exit 1;
fi

irq_list=(`cat /proc/interrupts | grep $1 | awk -F ':' '{print $1}'`)
start_core=$2
end_core=$3
for irq in ${irq_list[@]}
do
    echo $start_core > /proc/irq/$irq/smp_affinity_list
    printf "IRQ $irq @core "; echo `cat /proc/irq/$irq/smp_affinity_list`
    if [ "$start_core" == "$end_core" ] ; then
      start_core=$2
    else
      (( start_core+=1 ))
    fi
done
```
bindirq.sh脚本
```
#!/bin/bash
if [ $# != 5 ] ; then
    echo "USAGE: $0 eth_name  cpus_per_numa  numas  numa_node  thread_per_core"
    echo " e.g.: $0 enp130s0f0 44 4 0 2"
    exit 1;
fi

irq_list=(`cat /proc/interrupts | grep $1 | awk -F ':' '{print $1}'`)
start_core=0
end_core=$[$2/$5]
for irq in ${irq_list[@]}
do
    core_id=$[$start_core*$3+$4]
    echo $core_id > /proc/irq/$irq/smp_affinity_list
    #echo "$core_id > /proc/irq/$irq/smp_affinity_list"
    printf "IRQ $irq @core "; echo `cat /proc/irq/$irq/smp_affinity_list`
    #printf "IRQ $irq @core "; echo $core_id
    (( start_core+=1 ))
    if [ "$start_core" == "$end_core" ] ; then
      start_core=0
    fi
done
```

#### 网卡设置
开启网卡TSO、GSO特性，优化tcp协议通讯
```
ethtool -K <网卡名称> tso | gso on
```

调整网卡ring buffer配置
```
ethtool -g $eth                  #查看ring最大值，一般为4096，eth=<网卡名称>
ethtool -G $eth rx 4096 tx 4096  #调到最大值
```
多实例部署且多网卡时，也可将不同实例绑到不同网卡，在数据库配置文件中指定
bind_address=$IP

### 关闭透明大页
不建议对数据库服务器使用THP，在greatdbd_safe中对transparent huge pages和jemalloc进行了启动前检查，如未设置，按如下步骤修改。
查看设置
```
cat /sys/kernel/mm/transparent_hugepage/defrag
cat /sys/kernel/mm/transparent_hugepage/enabled
```

修改设置
```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
注意设置会重启失效

## 数据库参数优化
`innodb_buffer_pool_size`
设置为系统内存的50%~75%，多实例或主从场景可保证主的内存，降低从的内存，使buffer pool尽量可以缓存实例压测所需的全部数据，以提升读取速度
在多实例部署测试场景，可适当调高主节点buffer pool
MGR中由于认证数据库占用大量内存，该比例不再适用，可监测和修改，尤其是预置数据阶段

`innodb_buffer_pool_instances`
取值范围8~32，保证每个内存缓冲池大小(最好不低于1G)，提升并行内存读写，innodb_buffer_pool_size是innodb_buffer_pool_instances*innodb_buffer_pool_chunk_size的整数倍

`innodb_flush_sync`
设置为ON，数据库实例会根据自身需求自动调整，若想自定义设置io_capacity，需改为OFF
注：如果不是特别好的PCIE SSD建议用默认的ON就好。否则起多实例会造成性能波动更剧烈

`innodb_io_capacity`
一般设置为20000，设置为fio测试得到的iops，测试IOPS命令：
```
fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=15G -numjobs=20 -runtime=60 -group_reporting -name=mytest
```

`innodb_io_capacity_max`
一般设置为40000 ，取innodb_io_capacity的2倍值
注：io_capacity的经验值
NVME：15000-20000
SAS：200~400，
SSD：2000~4000，
PCIE：20000~40000
innodb_flush_method
版本5.7.25+以上支持，redo log和数据文件在同一存储设备上，并且存储设备有电池支持，可以改为O_DIRECT_NO_FSYNC（InnoDB 在刷新 I/O 时使用O_DIRECT，但在每次写操作后跳过 fsync ()系统调用）提高性能。
其余场景下，写性能瓶颈时，建议设置O_DSYNC。
含RAID卡和写缓存时，建议使用O_DIRECT帮助避免innodb缓冲池和os缓存之间的双缓冲；SAN文件系统及读场景多时，选用默认的fsync或o_dsync

`innodb_log_write_ahead_size`
一般设置为4096，设置redo的提前写块大小，该参数在MySQL5.7.4+引入，官方版本避免日志文件read-on-write的现象。默认8192，设置在 512~innodb_page_size之间，合法值是InnoDB log file block size (2n)的倍数。太小会引发read-on-write，太大会对fsync性能有轻微影响(因为同时刷多个block)

`innodb_flush_neighbors`
机械盘设置该值为1，即刷新缓冲区相邻脏页，SSD或PCI-E不用刷邻页，设置该值为0

`innodb_fsync_threshold`
该参数设定日志或表空间文件的从os缓存刷到磁盘的刷新字节值，默认0表示文件完全写入缓存后才刷盘，多实例共用磁盘时指定阈值强制小周期刷新会好一些，取值范围是0~2^64-1

`innodb_use_fdatasync`
在MySQL 8.0.26版本中，可以设置innodb_use_fdatasync=1，使用fdatasync()替换fsync()，可以避免不必要的linux inode磁盘写，对日志写优化明显。在支持fdatasync()的平台上可设置该参数为1来获得I/O性能提升。

`innodb_log_file_size`
`innodb_log_files_in_group`
写操作多可调大，但恢复时间长，一般设置：
innodb_log_file_size=1GB
innodb_log_files_in_group=4
性能测试可设置innodb_log_file_size=8G或更大值

`innodb_data_file_path`
设置innodb_data_file_path = ibdata1:1G:autoextend，默认值10M，在高并发事务场景下太小有性能影响

`innodb_flush_log_at_trx_commit`
0：不管有没有提交，每秒钟都写到binlog日志里。非生产环境，测试极限性能时设置
1：每次提交事务，都会把log buffer的内容写到磁盘里去，对日志文件做到磁盘刷新，安全性最好。
2：每次提交事务，都写到操作系统缓存，由OS刷新到磁盘。

`sync_binlog`
控制binlog刷盘频率，合法值为 0/1/N，测试极限性能可设置为0或较大数2000等

`innodb_doublewrite`
非生产环境，测试极限性能，可以设置为0，关闭双写避免额外I/O。

`table_open_cache`
mysqld打开表的数量 建议设置大一些，例如table_open_cache=30000。

`innodb_file_per_table`
innodb_file_per_table=1，独立表空间

`innodb_open_files`
在innodb_file_per_table=1模式下，限制Innodb能打开ibd文件数量。建议此值调大一些，尤其是表特别多的情况，例如innodb_open_files=2000

`innodb_thread_concurrency`
InnoDB使用操作系统线程来处理用户的事务请求。建议取默认值为0，它表示默认情况下不限制线程并发执行的数量。

`innodb_read_io_threads`
执行请求队列中的读请求操作的线程数。范围1-64，在实例上查看SHOW ENGINE INNODB STATUS的输出，如果有超过64 * innodb_read_io_threads个数量的读请求挂起，则需要调高该参数。

`innodb_write_io_threads`
执行请求队列中的写请求操作的线程数。范围1-64，该参数在设置过大时，在大量的写入请求分发操作系统时，可能会导致读I/O饥饿，需要与innodb_read_io_threads综合考量。

`ssl`
是否开启安全连接。安全连接对性能影响较大，非生产环境，测试极限性能，可以设置为0；生产环境根据客户需求调整。

`innodb_page_cleaners`
刷新脏数据的线程数。建议与innodb_buffer_pool_instances相等。

`table_open_cache_instances`
MySQL 缓存 table 句柄的分区的个数，建议设置16-32。

`table_open_cahce`
是所有线程打开的表数量，增加该值可以增加mysqld需要的文件描述符。对于是否需要增加该值，可以查看Opened_tables的状态。一个session在DML时会锁定一个instance，增加table_open_cache_instances可以减少session之间的争用。

`skip_log_bin`
是否开启binlog。在GreatDB 5.0非生产环境，测试极限性能在参数文件中增加此参数，关闭binlog选项（添加至配置文件中：skip_log_bin #log-bin=mysql-bin ）。

`slow_query_log`
slow_query_log=0 关闭慢查询，在I/O到达瓶颈时，避免慢SQL不必要落盘增加磁盘空间使用

`innodb_checksum_algorithm`
数据完整性校验。非生产环境，测试极限性能设置成innodb_checksum_algorithm=none，不启用算法校验。

`binlog_checksum`
Binlog完整性校验。非生产环境，测试极限性能设置成binlog_checksum=none，不启用算法校验。

`innodb_log_checksums`
Log完整性校验。非生产环境，测试极限性能设置成innodb_log_checksums=0，关闭log checksum。

`foreign_key_checks`
外键校验。非生产环境，测试极限性能设置成foreign_key_checks=0，关闭外键校验。

`performance_schema`
是否开启性能模式。非生产环境，测试极限性能设置为performance_schema=OFF，关闭性能模式。有10%的性能提升

`innodb_numa_interleave`
innodb_numa_interleave=off 关闭交错分配内存模式

`replication_optimize_for_static_plugin_config`
设置为on，如果基于8.0.23以后版本，打开此参数能在启用半同步时降低dml语句的全局锁竞争，对MGR无效



## 优化实践

```
双一参数关闭
半同步关闭 semi_sync
8.0 关闭redo log
innodb_doublewrite = 1		#改成innodb_doublewrite	=OFF
innodb_flush_log_at_trx_commit = 1 #改成0
innodb_flush_method = O_DIRECT_NO_FSYNC	  #改成DIRECT_NO_FSYNC
innodb_io_capacity = 8000
innodb_io_capacity_max = 10000
上边两个没用，需要开启手动需要innodb_flush_sync=OFF。去掉即可
innodb_print_all_deadlocks = on	@如果没要求记录所以死锁信息改成off
sync_binlog = 1  #改成0

innodb_buffer_pool_instances = 6		#改成8

新增
innodb_log_write_ahead_size =4096 			# MySQL5.7.4+引入了一个新参数，官方版本避免日志文件read-modify-write的现象。

		#刷新缓冲区相邻脏页
innodb_log_compressed_pages=0 	#使用InnoDB表压缩特性时，在对压缩数据进行更改时，重新压缩页的映像会写入redo log。

innodb_sync_spin_loops = 30


主库优化：
配置文件
innodb_buffer_pool  内存调大 200G
innodb_buffer_pool_instances = 20
log_file 8G
innodb_flush_method = O_DIRECT_NO_FSYNC	
innodb_flush_sync=OFF
innodb_log_write_ahead_size =4096 
innodb_log_compressed_pages=0 
innodb_sync_spin_loops = 30
innodb_print_all_deadlocks = 0

动态修改
dbscale backend server execute server execute "set global innodb_flush_log_at_trx_commit = 0;";
dbscale backend server execute server execute "set global sync_binlog = 0;";
dbscale execute on dataserver par_0_0 "set global semi_sync_master=0"
dbscale execute on dataserver par_1_0 "set global semi_sync_master=0"
dbscale execute on dataserver par_2_0 "set global semi_sync_master=0"
dbscale execute on dataserver par_3_0 "set global semi_sync_master=0"
dbscale execute on dataserver par_4_0 "set global semi_sync_master=0"

ALTER INSTANCE DISABLE INNODB REDO_LOG;

大并发开启线程池
innodb_sync_spin_loops 70
innodb_spin_wait_delay 120



海光 7285 CPU 一体机测试总结

1、线程池
大并发测试开启连接池有效降低cpu sys 使用率 10%左右，性能1000仓 tpmc 20w 提升至25w+ 左右（现场数据，仅供参考）
```
thread_handling=pool-of-threads
thread_pool_oversubscribe=3
thread_pool_size=96			（建议为逻辑CPU个数），现场可以根据测试并发进行调整测试，选择较优值
```
2、sysbench 压测过程中发现系统函数 ut_delay 占用高，此为cpu spin 相关参数
在海光CPU 7285 下实测调整如下参数有一定提升
```
innodb_sync_spin_loops 60
innodb_spin_wait_delay 120
```

3、其他优化点
监控关闭（performance_schema、innodb_monitor）
双一调整（高速SSD IO没有达到瓶颈时没有太大效果）
innodb_flush_sync=OFF  innodb_flush_sync参数（默认情况下是ON启用）会导致在检查点发生的I / O活动突发时忽略innodb_io_capacity设置。 要遵守innodb_io_capacity设置定义的InnoDB后台I / O活动限制，请禁用innodb_flush_sync。
innodb_io_capacity = 10000
innodb_io_capacity_max = 200000   （具体值建议对磁盘做16k 随机读写测试）
innodb_flush_method= 此次压测使用 O_DIRECT_NO_FSYNC。一般使用 O_DIRECT，压测试可以对比
innodb_log_file_size= 4G  4个
innodb_print_all_deadlocks = 0

4、关闭半同步

5、在未达到瓶颈时加大压力 CPU 到70% 左右无法继续提升
测试环境共有160G 内存，8个 numa node，比常规CPU 2个numa node更多，势必跨node 内存访问会更加频繁，响应时间更高
greatdbd 绑定部分CPU 负优化，性能更低，内存不足
iperf 测试带宽未达瓶颈
多sysbench 同时压测排除 sysbench 自身资源争用
 
6、greatdb 单机并行load
无主键表并行load 并发越高导入越慢，比单线程导入还慢
主要消耗在innodb 构建隐式主键时锁争用，并发越高锁争用越严重。

实测，无主键 20列，3kw 行数据，5.6G 文件
不开并行  5min30s
6 并发   7min20s
16并发   9min38s

加自增主键后数据导出再导入
6并发 2min 16s
16并发 1min 23s
```
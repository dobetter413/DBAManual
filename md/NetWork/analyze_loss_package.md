# 网络丢包分析

### 工具
#### 1、ping 
ping 基于 ICMP 协议,可以通过判断是否存在丢包情况.但是有的环境下icmp 协议禁用使用ping 无法分析和判断网络情况.
使用实例:
```
[root@cm-1 ~]# ping 172.17.140.99
PING 172.17.140.99 (172.17.140.99) 56(84) bytes of data.
64 bytes from 172.17.140.99: icmp_seq=1 ttl=64 time=0.039 ms
64 bytes from 172.17.140.99: icmp_seq=2 ttl=64 time=0.051 ms
64 bytes from 172.17.140.99: icmp_seq=3 ttl=64 time=0.081 ms
64 bytes from 172.17.140.99: icmp_seq=4 ttl=64 time=0.058 ms
64 bytes from 172.17.140.99: icmp_seq=5 ttl=64 time=0.037 ms
64 bytes from 172.17.140.99: icmp_seq=6 ttl=64 time=0.048 ms
^C
--- 172.17.140.99 ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 4999ms
rtt min/avg/max/mdev = 0.037/0.052/0.081/0.015 ms
```

#### 2、hping3
```
# -c表示发送10个请求，-S表示使用TCP SYN，-p指定端口为80
hping3 -c 10 -S -p 80 192.168.0.30
 
HPING 192.168.0.30 (eth0 192.168.0.30): S set, 40 headers + 0 data bytes
len=44 ip=192.168.0.30 ttl=63 DF id=0 sport=80 flags=SA seq=3 win=5120 rtt=7.5 ms
len=44 ip=192.168.0.30 ttl=63 DF id=0 sport=80 flags=SA seq=4 win=5120 rtt=7.4 ms
len=44 ip=192.168.0.30 ttl=63 DF id=0 sport=80 flags=SA seq=5 win=5120 rtt=3.3 ms
len=44 ip=192.168.0.30 ttl=63 DF id=0 sport=80 flags=SA seq=7 win=5120 rtt=3.0 ms
len=44 ip=192.168.0.30 ttl=63 DF id=0 sport=80 flags=SA seq=6 win=5120 rtt=3027.2 ms
 
--- 192.168.0.30 hping statistic ---
10 packets transmitted, 5 packets received, 50% packet loss
round-trip min/avg/max = 3.0/609.7/3027.2 ms
```

### 丢包链路分析
+ 在两台 VM 连接之间，可能会发生传输失败的错误，比如网络拥塞、线路错误等；
+ 在网卡收包后，环形缓冲区可能会因为溢出而丢包；
+ 在链路层，可能会因为网络帧校验失败、QoS 等而丢包；
+ 在 IP 层，可能会因为路由失败、组包大小超过 MTU 等而丢包；
+ 在传输层，可能会因为端口未监听、资源占用超过内核限制等而丢包；
+ 在套接字层，可能会因为套接字缓冲区溢出而丢包；
+ 在应用层，可能会因为应用程序异常而丢包；
+ 此外，如果配置了 iptables 规则，这些网络包也可能因为 iptables 过滤规则而丢包


### 链路层分析
当链路层由于缓冲区溢出等原因导致网卡丢包时，Linux 会在网卡收发数据的统计信息中记录下收发错误的次数。可以通过 ethtool 或者 netstat ，来查看网卡的丢包记录。

```
[root@cm-1 ~]# netstat -i
Kernel Interface table
Iface             MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0             1500 162932932      0      6 0       4593614      0      0      0 BMRU
lo              65536    42327      0      0 0         42327      0      0      0 LRU
virbr0           1500        0      0      0 0             0      0      0      0 BMU
You have new mail in /var/spool/mail/root
[root@cm-1 ~]# 
```

RX-OK、RX-ERR、RX-DRP、RX-OVR ，分别表示接收时的总包数、总错误数、进入 Ring Buffer 后因其他原因（如内存不足）导致的丢包数以及 Ring Buffer 溢出导致的丢包数。
TX-OK、TX-ERR、TX-DRP、TX-OVR 也代表类似的含义，只不过是指发送时对应的各个指标。


如果用 tc 等工具配置了 QoS，那么 tc 规则导致的丢包，就不会包含在网卡的统计信息中。所以接下来，我们还要检查一下 eth0 上是否配置了 tc 规则，并查看有没有丢包。添加 -s 选项，以输出统计信息：
```
tc -s qdisc show dev eth0
 
qdisc netem 800d: root refcnt 2 limit 1000 loss 30%
 Sent 432 bytes 8 pkt (dropped 4, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
```

### 网络层和传输层分析
在网络层和传输层中，引发丢包的因素非常多。不过，其实想确认是否丢包，是非常简单的事，因为 Linux 已经为我们提供了各个协议的收发汇总情况。执行 netstat -s 命令，可以看到协议的收发汇总，以及错误信息：
```
netstat -s
#输出
Ip:
    Forwarding: 1          //开启转发
    31 total packets received    //总收包数
    0 forwarded            //转发包数
    0 incoming packets discarded  //接收丢包数
    25 incoming packets delivered  //接收的数据包数
    15 requests sent out      //发出的数据包数
Icmp:
    0 ICMP messages received    //收到的ICMP包数
    0 input ICMP message failed    //收到ICMP失败数
    ICMP input histogram:
    0 ICMP messages sent      //ICMP发送数
    0 ICMP messages failed      //ICMP失败数
    ICMP output histogram:
Tcp:
    0 active connection openings  //主动连接数
    0 passive connection openings  //被动连接数
    11 failed connection attempts  //失败连接尝试数
    0 connection resets received  //接收的连接重置数
    0 connections established    //建立连接数
    25 segments received      //已接收报文数
    21 segments sent out      //已发送报文数
    4 segments retransmitted    //重传报文数
    0 bad segments received      //错误报文数
    0 resets sent          //发出的连接重置数
Udp:
    0 packets received
    ...
TcpExt:
    11 resets received for embryonic SYN_RECV sockets  //半连接重置数
    0 packet headers predicted
    TCPTimeouts: 7    //超时数
    TCPSynRetrans: 4  //SYN重传数
  ...
```
netstat 汇总了 IP、ICMP、TCP、UDP 等各种协议的收发统计信息。不过，我们的目的是排查丢包问题，所以这里主要观察的是错误数、丢包数以及重传数

### iptables 分析
除了网络层和传输层的各种协议，iptables 和内核的连接跟踪机制也可能会导致丢包。所以，这也是发生丢包问题时我们必须要排查的一个因素。
iptables基于 Netfilter 框架，通过一系列的规则，对网络数据包进行过滤（如防火墙）和修改（如 NAT）。这些 iptables 规则，统一管理在一系列的表中，包括 filter、nat、mangle（用于修改分组数据） 和 raw（用于原始数据包）等。而每张表又可以包括一系列的链，用于对 iptables 规则进行分组管理。

对于丢包问题来说，最大的可能就是被 filter 表中的规则给丢弃了。要弄清楚这一点，就需要我们确认，那些目标为 DROP 和 REJECT 等会弃包的规则，有没有被执行到。可以直接查询 DROP 和 REJECT 等规则的统计信息，看看是否为0。如果不是 0 ，再把相关的规则拎出来进行分析。

```
iptables -t filter -nvL
#输出
Chain INPUT (policy ACCEPT 25 packets, 1000 bytes)
 pkts bytes target     prot opt in     out     source               destination
    6   240 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.29999999981
 
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 
Chain OUTPUT (policy ACCEPT 15 packets, 660 bytes)
 pkts bytes target     prot opt in     out     source               destination
    6   264 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.29999999981
```

# tcp_tw_recycle 引发故障分析

## 故障现象
通过F5地址连接数据库集群偶发出现连接超时退出等情况，直接连接实际dbscale 地址无法复现。

## 分析过程
查看TCP的数据情况

登录服务器查看TCP的数据情况，发现有大量的TCP SYN包被丢弃，且数值一直在增长。
```
$ netstat -s | grep -i listen

1825 times the listen queue of a socket overflowed

2751213 SYNs to LISTEN sockets dropped
```

存在NAT的网络环境中，同时启用了net.ipv4.tcp_tw_recycle和net.ipv4.tcp_timestamps两个参数。

其中net.ipv4.tcp_timestamps默认开启，net.ipv4.tcp_tw_recycle需要手动开启。


主要原因：
1、time_wait出现在主动关闭tcp连接的一端（可能是服务端，也可能是客户端，以下简称本端，对应的另外一段称为对端），如果本端开启了tcp_tw_recycle，触发per-host PAWS机制，会检查对端IP维度的timestamps是否递增，如果不是递增的，会直接丢弃包（现象是例如发送SYN包，或者数据包对方没有回包，这边一直重试，直到多次重试都失败了，连接就会超时异常）。

2、但是这个超时问题只会出现在对端是NAT环境，互联网上大部分真实设备都是在NAT设备之下的，例如家庭多个手机/PC通过wifi上网，企业级的多个内网服务器通过防火墙/路由器访问公网。 但是在本端看来，都是同一个对端NAT IP连接过来的，而对端NAT环境下的多个设备时间肯定不会完全一致，存在差异，所以一旦在本端开启了net.ipv4.tcp_tw_recycle=1，对端发送过来的包携带的timestamp很难全部递增（无论是对端NAT环境下哪个设备发送的包，都认为是对端NAT IP的包），本端就会丢弃timestamps不递增的包，此时就可能导致对端的部分设备收不到回包，多次重传都失败连接就异常了。

3、大多数情况下，都是tcp连接上的客户端主动发起fin关闭，则time_wait状态会出现在客户端，而服务端不会出现time_wait状态，这种情况服务端是不需要考虑优化回收time_wait的。但是现代的大多数应用部署在服务器上，它即会作为服务端对外提供服务，也会作为客户端去请求其他服务器上的接口，前者因为是作为服务端，很少出现time_wait状态，后者作为客户端很容易在服务器上产生大量TCP time_wait状态，我们想快速回收time_wait，主要是针对后者这种情况。此时如果我们想解决后者time_wait过多的情况，在服务器（注意是服务器，不是服务端）上开启tcp_tw_recycle快速回收端口，因为这台服务器上的应用同时又是作为服务端对外提供服务，tcp_tw_recycle触发的pre-host PAWS机制，会使得客户端超时异常

所以net.ipv4.tcp_tw_recycle=1不能轻易开启，如果非要开启只能在以下两种场景：

1、高并发内部调用，一般不存在客户端proxy(即客户端NAT)
2、自己对上下游应用有较强的可控性，可以随时调整上下游的参数一起去适配
3、这台服务器上的应用，只会作为客户端频繁请求其他IP（一般情况都是客户端主动关闭TCP，产生time_wait状态），没有作为服务端对外提供服务。
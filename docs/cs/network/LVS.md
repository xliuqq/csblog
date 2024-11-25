# Linux Virtual Server

LVS的模型中有两个角色：

- 调度器：Director，又称为Dispatcher，Balancer
  - 调度器主要用于接受用户请求。

- 真实主机:Real Server，简称为RS。
  - 用于真正处理用户的请求。

Director Virtual IP：

- 调度器用于与客户端通信的IP地址，简称为 **VIP**

Director IP：

- 调度器用于与RealServer通信的IP地址，简称为DIP。

Real Server ：

- 后端主机的用于与调度器通信的IP地址，简称为RIP。

 

## VS/NAT技术

Virtual Server via Network Address Translation

通过网络地址转换（Network Address Translation）将一组服务器构成一个高性能的、高可用的虚拟服务器；

<img src=".pics/LVS/vs_nat.png" alt="img"  />

优点：  

- 支持端口映射，RS可以使用任意操作系统，节省公有IP地址；
- RIP和DIP都应该使用**同一网段私有地址**，而且RS的网关要指向DIP；
- 使用nat另外一个好处就是后端的主机相对比较安全。   

缺点：  

- 请求和响应报文都要经过Director转发，极高负载时，Director可能成为系统瓶颈。  



## VS/TUN 

Virtual Server via IP Tunneling：IP隧道实现虚拟服务器的方法；

IP隧道（IP tunneling）是**将一个IP报文封装在另一个IP报文**的技术；

- RealServer则会使用lo接口上的VIP直接响应CIP；
- VS/TUN技术对服务器有要求，即所有的服务器必须支持“IP Tunneling”或者“IP Encapsulation”协议；

![vs_tun.png](.pics/LVS/vs_tun.png)

优点：  

- **RIP，VIP，DIP都应该使用公网地址**，且RS网关不指向DIP;
- 只接受进站请求，解决了LVS-NAT时的问题，减少负载。  
- 请求报文经由Director调度，但是**响应报文不需经由Director**。

缺点：  

- 不指向Director所以不支持端口映射。 
- RS的OS必须支持隧道功能。 
- 隧道技术会额外花费性能，增大开销。



### VS/DR

Virtual Server via Direct Routing：通过直接路由实现虚拟服务器的方法；

- 目标地址的MAC地址改为RealServer的MAC地址。
- RealServer接受到转发而来的请求，发现目标地址是VIP。RealServer配置在lo接口上。
- 处理请求之后则**使用lo接口上的VIP**响应CIP。

![vsdr.png](.pics/LVS/vsdr.png)

优点： 

-  RIP可以**使用私有地址**，也可以使用公网地址；
-  只要求DIP和RIP的地址在同一个网段内；
-  请求报文经由Director调度，**但是响应报文不经由Director**；
-  RS可以使用大多数OS。

缺点：  

- 不支持端口映射，Director不能更改请求报文的端口；
- 不能跨局域网。



## 主备

LVS的容灾可以通过主备+心跳的方式实现。主LVS失去心跳后，备LVS可以作为热备立即替换。

容灾主要是靠KeepAlived来做的。

 

### ipvsadm 

ipvsadm -A -t     192.168.2.211:80 -s rr 

ipvsadm -a -t 192.168.2.211:80 -r 192.168.2.203 -g

ipvsadm -a -t 192.168.2.211:80 -r 192.168.2.204 -g

 

 

## RealServer

改RealServer内核参数

echo "1" > /proc/sys/net/ipv4/ip_forward

echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce

echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore

echo "1" > /proc/sys/net/ipv4/conf/lo/arp_ignore

echo "2" > /proc/sys/net/ipv4/conf/lo/arp_announce

ifconfig lo:0 192.168.2.211/32 broadcast 192.168.2.211 up

route add -host 192.168.2.211 dev lo:0

修改内核参数，并且配置VIP地址到RealServer的loopback接口上。

那样的话，当RealServer接到从Director转发而来的数据报文时，RealServer也不会丢弃报文。

同时，修改了RealServer的参数，局域网内的arp表就只有Director有VIP。

RealServer的的机器上有VIP这件事，只有RealServer自己知道。

这样可以保证，当请求到来的时候，第一个会送到Director那里去。
## calico网络策略

```
简介：calico完全通过三层网络实现路由转发
calico  ippool 以calico 3.2版本为准，在calico 3.3中增加了blockSize可变更以及
针对namespace单独设置子网的功能参考：https://www.projectcalico.org/calico-ipam-explained-and-enhanced/
calico不同于flannel，不需要为每个node分配子网段，所以只需要考虑pod的数量
例如配置 172.0.0.0/16 总计运行 2^16 = 65536个pod
```

### 1. 基本原理
完全通过三层网络的路由转发来实现

#### calico ippool
>以calico 3.2版本为准，在calico 3.3中增加了blockSize可变更以及针对namespace单独设置子网的功能
参考：https://www.projectcalico.org/calico-ipam-explained-and-enhanced/

calico 不同于flannel不需要为每个node分配子网段，所以只需要考虑pod的数量；

例如配置 172.0.0.0/16即总共可运行 2^16=65536个pod。

#### ippool分配细节
默认情况下，当节点网络中出现第一个容器，calico会为容器分配一段子网（子网掩码为26 例如：172.0.118.0/26），

后续出现该节点上的pod都从这个子网中分配ip地址，这样做可以缩减节点上路由表的规模，按照这种分配，

节点上 2^6=64个ip地址只需要一张路由表项就可以了，而不是为每个ip单独创建一个路由表项。

>注意：当64个主机位都用完后，会从其他可用的网段中取值，并不强制该节点只运行64个pod，只是增加的路由表项

#### calico的转发细节
容器A1的 IP 地址为 172.17.8.2/32，注意，不是 /24，而是 /32，将容器A1作为一个单点的局域网。

容器A1的默认路由为：
```
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0 scope link
```
IP地址为 169.254.1.1 ，是默认的网关，但是整个拓扑图没有一张网卡是这个地址。如何能到达这个地址呢？

ARP本地有缓存，通过 ip neigh 命令查看
```
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee STALE
```
找个mac地址为外面 caliefb22cb5e12 的mac地址，即到达物理机A路由器：
```
172.17.8.2 dev veth1 scope link 
172.17.8.3 dev veth2 scope link 
172.17.9.0/24 via 192.168.100.101 dev eth0 proto bird onlink
```
为了保证物理机A和物理机B能够相互知道，路由信息，calico使用了bgp协议进行路由发现。

#### IPIP
跨网段之间的访问，启用 IPIP 模式

使用了 IPIP 模式后，在物理机A上看到的路由表：
```
172.17.8.2 dev veth1 scope link 
172.17.8.3 dev veth2 scope link 
172.17.9.0/24 via 192.168.200.101 dev tun0 proto bird onlink
```
和原来模式的区别，下一跳不再是同一个网段的物理机B了，IP 为 192.168.200.101，

并且不是从 eth0 跳，而是建立一个隧道的端点 tun0，从这里才是下一跳。
```
内层源 IP 为 172.17.8.2；
内层目标 IP 为 172.17.9.2；
外层源 IP 为 192.168.100.100；
外层目标 IP 为 192.168.200.101。
```
将这个包从 eth0 发出去，在物理网络上会使用外层的 IP 进行路由，最终到达物理机 B。

在物理机 B 上，tun0 会解封装，将内层的源 IP 和目标 IP 拿出来，转发给相应的容器。

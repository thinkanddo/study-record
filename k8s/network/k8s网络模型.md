### k8s网络模型
>引用： https://developer.aliyun.com/article/745437?spm=a2c6h.14164896.0.0.12b257b9u0byyc
#### 名词
```
容器间的通信：同一个Pod内的多个容器间的通信，它们之间通过lo网卡进行通信。
Pod之间的通信：通过Pod IP地址进行通信。
Pod和Service之间的通信：Pod IP地址和Service IP进行通信，两者并不属于同一网络，实现方式是通过IPVS或iptables规则转发。
Service和集群外部客户端的通信，实现方式：Ingress、NodePort、Loadbalance
```
实现pod网络方案的方式： 虚拟网桥、多路复用（MacVLAN）、硬件交换（SR-IOV）

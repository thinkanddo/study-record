### calico网络策略

```
简介：calico完全通过三层网络实现路由转发
calico  ippool 以calico 3.2版本为准，在calico 3.3中增加了blockSize可变更以及针对namespace单独设置子网的功能参考：https://www.projectcalico.org/calico-ipam-explained-and-enhanced/
calico不同于flannel，不需要为每个node分配子网段，所以只需要考虑pod的数量
例如配置 172.0.0.0/16 总计运行 2*16 = 65536个pod
```

#### 1. 基本原理
完全通过三层网络的路由转发来实现

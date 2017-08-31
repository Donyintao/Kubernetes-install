# 搭建Flannel服务

## 什么是Flannel？
`Flannel`是`CoreOS`团队针对`Kubernetes`设计的一个覆盖网络(Overlay Network)工具，其目的在于帮助每一个使用`Kuberentes`的`CoreOS`主机拥有一个完整的子网。

## Flannel工作原理
`flannel`为全部的容器使用一个`network`，然后在每个`host`上从`network`中划分一个子网`subnet`。`host`上的容器创建网络时，从`subnet`中划分一个ip给容器。`flannel`不存在所谓的控制节点，而是每个`host`上的`flanneld`从一个etcd中获取相关数据，然后声明自己的子网网段，并记录在etcd中。如果有`host`对数据转发时，从`etcd`中查询到该子网所在的`host`的`ip`，然后将数据发往对应`host`上的`flanneld`，交由其进行转发。

## Flannel架构介绍
![flannel](file:///usr/local/src/tmp/init/Kubernetes-install/flannel.png)

# Kubernetes-install

本篇文章主要介绍kubernetes v1.13.x版本(启用TLS bootstrap认证方式)的安装步骤和注意事项；而不是使用kubeadm等自动化方式来部署集群。

安装步骤和流程，Kubernetes搭建部分参考文章：`和我一步步部署kubernetes集群`。主要修改：移除Token认证，增加了`Node TLS bootstrap`认证和使用`IPVS`实现Kubernetes入口流量负载均衡，以及`Master`高可用等；后面组件增加了`Calico`网络，`Traefik Ingress`，`Prometheus`监控报警，日志搜集等等；以及简单说明各个组件和服务的原理和作用。

本篇文章主要包含几部分：Kubernetes ipvs模式，CoreDNS，监控报警，日志搜集，docker私有仓库等等一系列解决方案；安装过程中，可能不会详细说明各组件的启动参数和作用；启动参数的详细介绍请参考其他的优秀博客和官方文档。


所有服务的搭建过程均在CentOS7，内核版本：4.4.xx系统上操作通过，其他系统未验证，有任何问题欢迎反馈。

## 环境准备

### 系统环境

| 主机名称   |  IP地址      | 系统        | 内核                          | 节点角色            |
| :-------: | :----------: | :---------: | :-------------------------: | :-----------------: |
| k8s-node1 | 172.16.0.101 | CentOS-7.4  | 4.4.166-1.el7.elrepo.x86_64 | Matser, Node, Etcd  | 
| k8s-node2 | 172.16.0.102 | CentOS-7.4  | 4.4.166-1.el7.elrepo.x86_64 | Node, Etcd          |
| k8s-node3 | 172.16.0.103 | CentOS-7.4  | 4.4.166-1.el7.elrepo.x86_64 | Node, Etcd          |
| haproxy   | 172.16.0.104 | CentOS-7.4  | 4.4.166-1.el7.elrepo.x86_64 | Haproxy, Keepalived |
| haproxy   | 172.16.0.105 | CentOS-7.4  | 4.4.166-1.el7.elrepo.x86_64 | Haproxy, Keepalived |

### 网络规划

| 网络类型  |  地址网段      |
| :-----: | :----------: |
| 物理网段 | 172.16.0.0/24 |
| 容器网段 | 10.240.0.0/16 |
| SVC网段 | 10.241.0.0/16 |

## kube-ansible快速部署
新增基于`ansible-playbook`方式实现的自动化部署`kubernetes`高可用集群环境，具体流程参考--[kube-ansible快速部署](https://github.com/Donyintao/kube-ansible)

## 手动安装步骤流程

1. [CA证书和秘钥](创建TLS证书和秘钥.md)
1. [Etcd集群服务](部署Etcd集群服务.md)
1. [网络方案选型]()
    1. [Flannel](部署Flannel服务.md)
    1. [Calico](部署Calico服务.md)
1. [Haproxy服务](部署Haproxy服务.md)
    1. [Master高可用服务之Haproxy服务](部署Haproxy服务.md)
1. [Kubrnetes服务](https://github.com/Donyintao/Kubernetes-install)
    1. [Master install](部署Kubrnetes-Master节点.md)
    1. [Node install](部署Kubrnetes-Node节点.md)
1. [Kubrnetes组件](https://github.com/Donyintao/Kubernetes-install)
   1. [CoreDNS](部署CoreDNS服务.md)
   1. [Dashboard](https://github.com/Donyintao/kubernetes-dashboard/)
1. [Ingress controllers](https://github.com/Donyintao/Kubernetes-install)
   1. [Ingress nginx](https://github.com/Donyintao/nginx-ingress/)
   2. [Ingress traefik](https://github.com/Donyintao/traefik/)
1. [Kubrnetes监控](https://github.com/Donyintao/Kubernetes-install)
    1. [Prometheus](https://github.com/Donyintao/Prometheus/)
    1. [Grafana](https://github.com/Donyintao/Grafana/)
    1. [Alertmanager](https://github.com/Donyintao/Alertmanager/)
1. [后续相关组件待补充......](后续相关组件待补充.md)

# Kubernetes-install

本篇文章主要介绍kubernetes v1.7.x版本(启用TLS方式)的安装步骤和注意事项，而不是使用kubeadm等自动化方式来部署集群。

安装步骤和流程，主要参考<<我一步步部署kubernetes集群>>文章，主要改动：去除了token认证，增加了Node TLS认证，简单说明各个组件的作用和用途。

本篇文章在搭建的过程中，不会详细说明各组件的启动参数和作用，具体启动参数详细介绍请参考其他的优秀博客和官方文档。

所有搭建过程均在CentOS7，内核版本：3.10.xx系统上操作，其他系统未验证，有任何问题欢迎反馈。

## 步骤列表

1. [建TLS证书和秘钥](创建TLS证书和秘钥.md)
1. [搭建etcd集群服务](搭建etcd集群服务.md)
1. [搭建flannel服务](搭建flannel服务.md)
1. [搭建Kubrnetes-Master节点](搭建Kubrnetes-Master节点.md)
1. [搭建Kubrnetes-Node节点](搭建Kubrnetes-Node节点.md)
1. [后续相关组件待补充......](后续相关组件待补充.md)

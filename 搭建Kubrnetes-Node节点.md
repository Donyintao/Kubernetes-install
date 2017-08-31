# 部署Kubernetes Node节点

## kubelet组件

每个节点上都运行一个kubelet服务进程，默认监听10250端口，接收并执行master发来的指令，管理Pod及Pod中的容器。每个kubelet进程会在API Server上注册节点自身信息，定期向master节点汇报节点的资源使用情况，并通过cAdvisor监控节点和容器的资源。

## kubelet管理

节点管理主要是节点自注册和节点状态更新：

+ Kubelet可以通过设置启动参数 --register-node 来确定是否向API Server注册自己；
+ 如果Kubelet没有选择自注册模式，则需要用户自己配置Node资源信息，同时需要告知Kubelet集群上的API Server的位置；
+ Kubelet在启动时通过API Server注册节点信息，并定时向API Server发送节点新消息，API Server在接收到新消息后，将信息写入etcd。

## kube-proxy组件

每台机器上都运行一个kube-proxy服务，它监听API server中service和endpoint的变化情况，并通过iptables等来为服务配置负载均衡(仅支持TCP和UDP)。
kube-proxy可以直接运行在物理机上，也可以以static pod或者daemonset的方式运行。

## kube-proxy实现方式

当前kube-proxy支持三种种实现方式：

+ userspace：最早的负载均衡方案，它在用户空间监听一个端口，所有服务通过iptables转发到这个端口，然后在其内部负载均衡到实际的Pod。该方式最主要的问题是效率低，有明显的性能瓶颈。
+ iptables：目前推荐的方案，完全以iptables规则的方式来实现service负载均衡。该方式最主要的问题是在服务多的时候产生太多的iptables规则(社区有人提到过几万条)，大规模下也有性能问题
+ winuserspace：同userspace，但仅工作在windows上

## 下载kubernetes组件的二进制文件

``` bash
# wget https://storage.googleapis.com/kubernetes-release/release/v1.7.4/kubernetes-server-linux-amd64.tar.gz
# tar fx kubernetes-server-linux-amd64.tar.gz
```

拷贝二进制文件

``` bash
# mkdir -p /usr/local/kubernetes-v1.7.4/bin
# ln -s /usr/local/kubernetes-v1.7.4 /usr/local/kubernetes
# cp -r `pwd`/kubernetes/server/bin/{kube-proxy,kubelet} /usr/local/kubernetes/bin
```

## 配置kube-config文件

``` bash
# vim /etc/kubernetes/kube-config
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=false"
 
# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"
 
# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"
 
# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=https://192.168.100.110:6443"

## 配置和启动kubelet服务

创建kubelet配置文件

``` bash
# vim /etc/kubernetes/kubelet
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=192.168.100.110"
#
## The port for the info server to serve on
KUBELET_PORT="--port=10250"
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=192.168.100.110"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.0"
#
## Add your own!
KUBELET_ARGS="--cgroup-driver=systemd \
              --cluster-dns=172.16.0.2 \
              --require-kubeconfig=true \
              --serialize-image-pulls=false \
              --cluster-domain=cluster.local. \
              --hairpin-mode promiscuous-bridge \
              --log-dir=/data/kubernetes/logs/kubelet \
              --client-ca-file=/etc/kubernetes/ssl/ca.pem \
              --tls-cert-file=/etc/kubernetes/ssl/kubelet.pem \
              --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
              --tls-private-key-file=/etc/kubernetes/ssl/kubelet-key.pem"
```

创建kubelet TLS认证配置文件
注意：1.7.x版本中增加了Node Restriction模式，采用system:node:NodeName方式来认证

``` bash
# vim /etc/kubernetes/kubelet.kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://192.168.100.110:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:node:node10
  name: kube-context
current-context: kube-context
users:
- name: system:node
  user:
    client-certificate: /etc/kubernetes/ssl/kubelet.pem
    client-key: /etc/kubernetes/ssl/kubelet-key.pem
```

创建kubelet启动脚本

``` bash
# vim /usr/lib/systemd/system/kubelet.service 
[Unit]
Description=Kubelet Service
After=network.target
 
[Service]
WorkingDirectory=/data/kubelet
EnvironmentFile=-/etc/kubernetes/kube-config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/kubernetes/bin/kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_API_SERVER \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_POD_INFRA_CONTAINER \
            $KUBELET_ARGS
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```

创建kubelet数据目录和日志目录

``` bash
# mkdir /data/kubelet -p
# mkdir /data/kubernetes/logs/kubelet -p
# systemctl daemon-reload
# systemctl enable kubelet && systemctl start kubelet && systemctl status kubelet
```

## 配置和启动kube-proxy服务

创建kube-proxy配置文件

``` bash
# vim /etc/kubernetes/kube-proxy
###
# kubernetes proxy config
 
# default config should be adequate
 
# Add your own!
KUBE_PROXY_ARGS="--bind-address=192.168.100.114 \
                 --hostname-override=192.168.100.114 \
                 --cluster-cidr=172.16.0.0/16 \
                 --log-dir=/data/kubernetes/logs/kube-proxy \
                 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig"
```

创建kube-proxy TLS认证配置文件

``` bash
# vim /etc/kubernetes/kube-proxy.kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://192.168.100.110:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-proxy
  name: kube-context
current-context: kube-context
users:
- name: system:kube-proxy
  user:
    client-certificate: /etc/kubernetes/ssl/kube-proxy.pem
    client-key: /etc/kubernetes/ssl/kube-proxy-key.pem
```

创建kube-proxy启动脚本

``` bash
# vim /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kube-proxy Service
After=network.target
 
[Service]
EnvironmentFile=-/etc/kubernetes/kube-config
EnvironmentFile=-/etc/kubernetes/kube-proxy
ExecStart=/usr/local/kubernetes/server/bin/kube-proxy \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_PROXY_ARGS
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```
 
创建kube-proxy日志目录

# mkdir /data/kubernetes/logs/kube-proxy -p
# systemctl daemon-reload
# systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy

## 验证Node节点是否正常

注意: 验证输出结果为某测试环境节点信息，输出`STATUS`为`Ready`说明正常。可以自行创建Pod验证

``` bash
# kubectl get node -o wide
NAME           STATUS    AGE       VERSION   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION
192.168.3.95   Ready     34d       v1.7.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.el7.x86_64
192.168.3.96   Ready     34d       v1.7.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.el7.x86_64
192.168.3.99   Ready     34d       v1.7.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.el7.x86_64
```

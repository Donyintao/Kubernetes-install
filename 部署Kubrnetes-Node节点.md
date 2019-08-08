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

当前kube-proxy支持四种实现方式：

+ userspace：最早的负载均衡方案，它在用户空间监听一个端口，所有服务通过iptables转发到这个端口，然后在其内部负载均衡到实际的Pod。该方式最主要的问题是效率低，有明显的性能瓶颈。
+ iptables：目前推荐的方案，完全以iptables规则的方式来实现service负载均衡。该方式最主要的问题是在服务多的时候产生太多的iptables规则(社区有人提到过几万条)，大规模下也有性能问题
+ winuserspace：同userspace，但仅工作在windows上
+ ipvs：1.8版本以后引入了ipvs模式，目前最新版本为GA版本

## 升级master和node节点内核版本
``` bash
# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
# yum -y install --enablerepo=elrepo-kernel kernel-lt-devel kernel-lt  
# grub2-set-default 0
# grub2-mkconfig -o /boot/grub2/grub.cfg
# reboot
# uname -r
4.4.112-1.el7.elrepo.x86_64
```

## 下载kubernetes组件的二进制文件

``` bash
# wget https://storage.googleapis.com/kubernetes-release/release/v1.15.2/kubernetes-server-linux-amd64.tar.gz
# tar fx kubernetes-server-linux-amd64.tar.gz
```

拷贝二进制文件

``` bash
# mkdir -p /usr/local/kubernetes-v1.15.2/bin
# ln -s /usr/local/kubernetes-v1.15.2 /usr/local/kubernetes
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
###
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=false"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=1"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=https://172.16.0.253:8443"
```

## 配置和启动kubelet服务

创建kubelet文件

``` bash
# vim /etc/kubernetes/kubelet
####
## kubernetes kubelet config
####
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=k8s-node1"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=k8s.gcr.io/pause:3.1"
#
## Add your own!
KUBELET_ARGS="--root-dir=/data/kubelet \
              --log-dir=/data/logs/kubernetes/kubelet \
              --cert-dir=/etc/kubernetes/ssl \
              --config=/etc/kubernetes/kubelet.config \
              --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
```

创建kubelet config文件

说明：1.12.x版本后，kubelet服务建议使用config方式配置

``` bash
# vim /etc/kubernetes/kubelet.config
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 172.16.30.0.101
port: 10250
cgroupDriver: cgroupfs
clusterDNS:
- 10.241.0.254
clusterDomain: testing.com.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
maxOpenFiles: 1000000
maxPods: 200
serializeImagePulls: true
failSwapOn: false
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
resolvConf: /etc/resolv.conf
runtimeRequestTimeout: 2m0s
```

创建kubelet TLS认证配置文件

注意：1.7.x以上版本中增加了Node Restriction模式，采用system:node:NodeName方式来认证

``` bash
# vim /etc/kubernetes/kubelet.kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://172.16.30.0.101:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:node:k8s-node1
  name: kube-context
current-context: kube-context
users:
- name: system:node:k8s-node1
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
            $KUBELET_HOSTNAME \
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
# mkdir /data/logs/kubernetes/kubelet -p
# systemctl daemon-reload
# systemctl enable kubelet && systemctl start kubelet && systemctl status kubelet
```

## 内核模块加载
``` bash
# yum -y install conntrack-tools ipvsadm ipset
# cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack_ipv4"
for kernel_module in \${ipvs_modules}; do
    /sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        /sbin/modprobe \${kernel_module}
    fi
done
EOF
# chmod +x /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
ip_vs_ftp              16384  0 
ip_vs_sed              16384  0 
ip_vs_nq               16384  0 
ip_vs_fo               16384  0 
ip_vs_sh               16384  0 
ip_vs_dh               16384  0 
ip_vs_lblcr            16384  0 
ip_vs_lblc             16384  0 
ip_vs_wrr              16384  0 
ip_vs_rr               16384  0 
ip_vs_wlc              16384  0 
ip_vs_lc               16384  0 
ip_vs                 147456  24 ip_vs_dh,ip_vs_fo,ip_vs_lc,ip_vs_nq,ip_vs_rr,ip_vs_sh,ip_vs_ftp,ip_vs_sed,ip_vs_wlc,ip_vs_wrr,ip_vs_lblcr,ip_vs_lblc
nf_nat                 28672  3 ip_vs_ftp,nf_nat_ipv4,nf_nat_masquerade_ipv4
nf_conntrack          110592  7 ip_vs,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_ipv4
libcrc32c              16384  2 xfs,ip_vs
```

## 配置和启动kube-proxy服务

创建kube-proxy配置文件

注意：1.12.x版本ipvs默认已经开启，可以启用`--proxy-mode=ipvs`参数开启ipvs.

``` bash
# vim /etc/kubernetes/kube-proxy
####
## kubernetes proxy config
####
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBE_PROXY_ADDRESS="--bind-address=172.16.30.0.101"
#
## You may leave this blank to use the actual hostname
KUBE_PROXY_HOSTNAME="--hostname-override=k8s-node1"
#
## Add your own!
KUBE_PROXY_ARGS="--proxy-mode=ipvs \
                 --ipvs-scheduler=rr \
                 --ipvs-sync-period=5s \
                 --ipvs-min-sync-period=5s \
                 --config=/etc/kubernetes/kube-proxy.config \
                 --log-dir=/data/logs/kubernetes/kube-proxy"
```

创建kube-proxy config文件

说明：1.12.x版本后，kube-proxy服务建议使用config方式配置

``` bash
# vim /etc/kubernetes/kube-proxy.config 
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 172.16.0.101
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 10.240.0.0/16
configSyncPeriod: 15m0s
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: ""
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
resourceContainer: /kube-proxy
udpIdleTimeout: 250ms
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
```

创建kube-proxy TLS认证配置文件

``` bash
# vim /etc/kubernetes/kube-proxy.kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
    server: https://172.16.30.0.101:6443
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
ExecStart=/usr/local/kubernetes/bin/kube-proxy \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_PROXY_ADDRESS \
        $KUBE_PROXY_HOSTNAME \
        $KUBE_PROXY_ARGS
 
Restart=on-failure
LimitNOFILE=65536
 
[Install]
WantedBy=multi-user.target
```
 
创建kube-proxy日志目录

``` bash
# mkdir /data/logs/kubernetes/kube-proxy -p
# systemctl daemon-reload
# systemctl enable kube-proxy && systemctl start kube-proxy && systemctl status kube-proxy
```


## 验证Node节点是否正常

注意: 验证输出结果为某测试环境节点信息，输出`STATUS`为`Ready`说明正常。可以自行创建Pod验证

``` bash
# kubectl get node -o wide
NAME        STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
k8s-node1   Ready     <none>    2d        v1.13.4    <none>        CentOS Linux 7 (Core)   4.4.166-1.el7.elrepo.x86_64   docker://17.9.1
k8s-node2   Ready     <none>    2d        v1.13.4    <none>        CentOS Linux 7 (Core)   4.4.166-1.el7.elrepo.x86_64   docker://17.9.1
k8s-node3   Ready     <none>    2d        v1.13.4    <none>        CentOS Linux 7 (Core)   4.4.166-1.el7.elrepo.x86_64   docker://17.9.1
```

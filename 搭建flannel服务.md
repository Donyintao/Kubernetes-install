# 搭建Flannel服务

## 什么是Flannel？
`Flannel`是`CoreOS`团队针对`Kubernetes`设计的一个覆盖网络(Overlay Network)工具，其目的在于帮助每一个使用`Kuberentes`的`CoreOS`主机拥有一个完整的子网。

## Flannel工作原理
`flannel`为全部的容器使用一个`network`，然后在每个`host`上从`network`中划分一个子网`subnet`。`host`上的容器创建网络时，从`subnet`中划分一个ip给容器。`flannel`不存在所谓的控制节点，而是每个`host`上的`flanneld`从一个etcd中获取相关数据，然后声明自己的子网网段，并记录在etcd中。如果有`host`对数据转发时，从`etcd`中查询到该子网所在的`host`的`ip`，然后将数据发往对应`host`上的`flanneld`，交由其进行转发。

## Flannel架构介绍

![](https://github.com/coreos/flannel/blob/master/packet-01.png)

## 创建Pod Network

注意: flanneld v0.7.1版本目前不支持etcd v3, 使用etcd v2 API写入配置key和网段数据

``` bash
# etcdctl set /flannel/network/config '{ "Network": "172.16.0.0/16", "Backend": { "Type": "host-gw" } }'
```

## 安装flannel服务

``` bash
# yum -y install flannel
# vim /etc/sysconfig/flanneld
# Flanneld configuration options  
 
# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://192.168.100.110:2379,http://192.168.100.111:2379,http://192.168.100.112:2379"
 
# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/flannel/network"
 
# Any additional options that you want to pass
FLANNEL_OPTIONS="-iface=eth0 -ip-masq"

# systemctl enable flanneld && systemctl restart flanneld && systemctl status flanneld
```

## 配置docker启动服务

注意: flannel服务要优先启动，docker服务启动脚本没有配置flannel服务优先级。

``` bash
# vim/usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target
After=flanneld.service
Wants=docker-storage-setup.service
Requires=docker-cleanup.timer

[Service]
Type=notify
NotifyAccess=all
KillMode=process
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
Environment=GOTRACEBACK=crash
Environment=DOCKER_HTTP_HOST_COMPAT=1
Environment=PATH=/usr/libexec/docker:/usr/bin:/usr/sbin
ExecStart=/usr/bin/dockerd-current \
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
Restart=on-abnormal
MountFlags=slave

[Install]
WantedBy=multi-user.target

# systemctl enable docker && systemctl restart docker && systemctl status docker
```
## 验证docker服务获取IP是否正常

注意：`yum`方式安装，docker和flannel无需做任何配置的；如果没有正确获取IP，请见flannel服务是否正常启动。

``` bash
# ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.36.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:d0:0b:23:be  txqueuelen 0  (Ethernet)
        RX packets 39657261  bytes 7409081483 (6.9 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 40524935  bytes 23758435104 (22.1 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

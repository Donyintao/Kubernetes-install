# 搭建Flannel服务

## 什么是Flannel？
`Flannel`是`CoreOS`团队针对`Kubernetes`设计的一个覆盖网络(Overlay Network)工具，其目的在于帮助每一个使用`Kuberentes`的`CoreOS`主机拥有一个完整的子网。

## Flannel工作原理
`flannel`为全部的容器使用一个`network`，然后在每个`host`上从`network`中划分一个子网`subnet`。`host`上的容器创建网络时，从`subnet`中划分一个ip给容器。`flannel`不存在所谓的控制节点，而是每个`host`上的`flanneld`从一个etcd中获取相关数据，然后声明自己的子网网段，并记录在etcd中。如果有`host`对数据转发时，从`etcd`中查询到该子网所在的`host`的`ip`，然后将数据发往对应`host`上的`flanneld`，交由其进行转发。

## Flannel架构介绍

![](https://github.com/coreos/flannel/blob/master/packet-01.png)

## 创建Pod Network

注意：flanneld v0.10.0版本目前不支持etcd v3, 使用etcd v2 API写入配置key和网段数据

注意：集群网段地址`172.20.0.0/16`, SVC(DNS)网段地址`172.21.0.0/16`

``` bash
# etcdctl set /flannel/network/config '{ "Network": "172.20.0.0/16", "Backend": { "Type": "host-gw" } }'
```

## 安装flannel服务

``` bash
# yum -y install flannel
# vim /etc/sysconfig/flanneld
# Flanneld configuration options  
 
# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://172.16.30.171:2379,http://172.16.30.172:2379,http://172.16.30.173:2379"
 
# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/flannel/network"
 
# Any additional options that you want to pass
FLANNEL_OPTIONS="-iface=eth0 -ip-masq"

# systemctl enable flanneld && systemctl restart flanneld && systemctl status flanneld
```

## 安装Docker CE服务
``` bash
# wget -P /etc/yum.repos.d/ https://download.docker.com/linux/centos/docker-ce.repo
# yum list docker-ce.x86_64  --showduplicates | sort -r
# yum -y install docker-ce-17.09.1.ce-1.el7.centos.x86_64
```
## 配置docker启动服务
vim /etc/docker/daemon.json
注意：flannel服务要优先启动，docker服务启动脚本没有配置flannel服务优先级。

``` bash
# vim /etc/docker/daemon.json
{
    "registry-mirrors": ["http://d7eabb7d.m.daocloud.io"],
    "storage-driver": "overlay2",
    "graph": "/data/docker"
}

# vim/usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service flanneld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target

# systemctl enable docker && systemctl restart docker && systemctl status docker
```
## 验证Docker服务获取IP是否正常

注意：如果Docker服务没有正确获取IP，Docker启动脚本是否配置`$DOCKER_NETWORK_OPTIONS`参数；并请检查flannel服务是否正常启动。

``` bash
# ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.20.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:d0:0b:23:be  txqueuelen 0  (Ethernet)
        RX packets 39657261  bytes 7409081483 (6.9 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 40524935  bytes 23758435104 (22.1 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

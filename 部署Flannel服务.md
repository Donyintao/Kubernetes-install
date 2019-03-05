# 搭建Flannel服务

## 什么是Flannel？
`Flannel`是`CoreOS`团队针对`Kubernetes`设计的一个覆盖网络(Overlay Network)工具，其目的在于帮助每一个使用`Kuberentes`的`CoreOS`主机拥有一个完整的子网。

## Flannel工作原理
`flannel`为全部的容器使用一个`network`，然后在每个`host`上从`network`中划分一个子网`subnet`。`host`上的容器创建网络时，从`subnet`中划分一个ip给容器。`flannel`不存在所谓的控制节点，而是每个`host`上的`flanneld`从一个etcd中获取相关数据，然后声明自己的子网网段，并记录在etcd中。如果有`host`对数据转发时，从`etcd`中查询到该子网所在的`host`的`ip`，然后将数据发往对应`host`上的`flanneld`，交由其进行转发。

## Flannel架构介绍

![](https://github.com/coreos/flannel/blob/master/packet-01.png)

## 创建Flanneld证书
``` bash
# cd /tmp/sslTmp
# cat > flanneld-csr.json << EOF
{
    "CN": "flanneld",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

生成Flanneld证书和私钥

``` bash
# cfssl gencert -ca=ca.pem \
                -ca-key=ca-key.pem \
                -config=ca-config.json \
                -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
```

## 创建Pod Network

注意：flanneld v0.11.0版本目前不支持etcd v3, 使用etcd v2 API写入配置key和网段数据

注意：容器网段地址`10.240.0/16`, SVC(DNS)网段地址`10.241.0.0/16`

``` bash
# mkdir -p /etc/flannel/ssl
# cp /tmp/sslTmp/{ca.pem,flanneld.pem,flanneld-key.pem} /etc/flannel/ssl
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/flannel/ssl/ca.pem \
          --cert-file=/etc/flannel/ssl/flanneld.pem \
          --key-file=/etc/flannel/ssl/flanneld-key.pem \
          set /flannel/network/config '{ "Network": "10.240.0.0/16", "Backend": { "Type": "host-gw" } }'
```

## 安装flannel服务

``` bash
# yum -y install flannel
# vim /etc/sysconfig/flanneld
# Flanneld configuration options  
 
# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://172.16.0.101:2379,https://172.16.0.102:2379,https://172.16.0.103:2379"
 
# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/flannel/network"
 
# Any additional options that you want to pass
FLANNEL_OPTIONS="-etcd-cafile=/etc/flannel/ssl/ca.pem -etcd-certfile=/etc/flannel/ssl/flanneld.pem -etcd-keyfile=/etc/flannel/ssl/flanneld-key.pem -iface=eth0 -ip-masq"

# systemctl enable flanneld && systemctl restart flanneld && systemctl status flanneld
```

## 安装Docker CE服务
``` bash
# wget -P /etc/yum.repos.d/ https://download.docker.com/linux/centos/docker-ce.repo
# yum list docker-ce.x86_64  --showduplicates | sort -r
# yum -y install docker-ce-17.09.1.ce-1.el7.centos.x86_64
```
## 配置docker启动服务

``` bash
# vim /etc/docker/daemon.json
{
    "registry-mirrors": ["http://d7eabb7d.m.daocloud.io"],
    "storage-driver": "overlay2",
    "graph": "/data/docker"
}

# vim /usr/lib/systemd/system/docker.service
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

注意：如果Docker服务没有获取正确的IP地址，请检查Docker启动脚本是否配置`$DOCKER_NETWORK_OPTIONS`参数；并检查flannel服务是否正常启动。

``` bash
# ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.240.36.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:d0:0b:23:be  txqueuelen 0  (Ethernet)
        RX packets 39657261  bytes 7409081483 (6.9 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 40524935  bytes 23758435104 (22.1 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

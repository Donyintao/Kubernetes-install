# 搭建etcd集群服务

## Etcd服务应用场景

要问`etcd`是什么？很多人第一反应可能是一个键值存储仓库，却没有重视官方定义的后半句，用于配置共享和服务发现。`etcd`作为一个受到`ZooKeeper`与`doozer`启发而催生的项目，除了拥有与之类似的功能外，更专注于以下四点:

+ 简单：基于HTTP+JSON的API让你用curl就可以轻松使用。
+ 安全：可选SSL客户认证机制。
+ 快速：每个实例每秒支持一千次写操作。
+ 可信：使用Raft算法充分实现了分布式。

分布式系统中的数据分为控制数据和应用数据。etcd的使用场景默认处理的数据都是控制数据，对于应用数据，只推荐数据量很小，但是更新访问频繁的情况。应用场景有如下几类: 

+ 场景一：服务发现（Service Discovery）
+ 场景二：消息发布与订阅
+ 场景三：负载均衡
+ 场景四：分布式通知与协调
+ 场景五：分布式锁、分布式队列
+ 场景六：集群监控与Leader竞选

举个最简单的例子，如果你需要一个分布式存储仓库来存储配置信息，并且希望这个仓库读写速度快、支持高可用、部署简单、支持http接口，那么就可以使用etcd。

## 创建Etcd证书
``` bash
# cat > etcd-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
      "127.0.0.1",
      "172.16.0.101",
      "172.16.0.102",
      "172.16.0.103"
    ],
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

生成Etcd证书和私钥

``` bash
# cfssl gencert -ca=ca.pem \
                -ca-key=ca-key.pem \
                -config=ca-config.json \
                -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

## 安装Etcd服务

``` bash
# yum -y install etcd
# cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak_$(date +%Y%m%d)
```

## 配置Etcd服务

``` bash
# 拷贝Etcd证书
# mkdir /etc/etcd/ssl
# cp /tmp/sslTmp/ca*.pem /etc/etcd/ssl
# cp /tmp/sslTmp/etcd*.pem /etc/etcd/ssl
# chown -R etcd: /etc/etcd/ssl
# 配置ETCD服务
# vim /etc/etcd/etcd.conf
ETCD_NAME=etcd_node1  // 节点名称
ETCD_DATA_DIR="/var/lib/etcd/etcd_node1.etcd"
ETCD_LISTEN_PEER_URLS="https://172.16.0.101:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.16.0.101:2379,https://127.0.0.1:2379"  // 必须增加127.0.0.1否则启动会报错
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.0.101:2380"
ETCD_INITIAL_CLUSTER="etcd_node1=https://172.16.0.101:2380,etcd_node2=https://172.16.0.102:2380,etcd_node3=https://172.16.0.103:2380"  // 集群IP地址
ETCD_INITIAL_CLUSTER_STATE="new"  // 初始化集群,第二次启动时将状态改为: "existing"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.0.101:2379"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_PEER_AUTO_TLS="true"
# systemctl enable etcd.service 
# systemctl start etcd.service && systemctl status etcd.service
```

设置API版本 默认版本：v2 [根据实际情况设定]

``` bash
# echo 'export ETCDCTL_API=3' >> /etc/profile
# . /etc/profile
```

## 查看和验证etcd集群服务状态

查看etcd集群成员

``` bash
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/etcd/ssl/ca.pem \
          --cert-file=/etc/etcd/ssl/etcd.pem \
          --key-file=/etc/etcd/ssl/etcd-key.pem member list 
7e218077496bccf9: name=etcd_node1 peerURLs=https://172.16.0.101:2380 clientURLs=https://172.16.0.101:2379 isLeader=true
92f1b7c038a4300a: name=etcd_node2 peerURLs=https://172.16.0.102:2380 clientURLs=https://172.16.0.102:2379 isLeader=false
c8611e11b142e510: name=etcd_node3 peerURLs=https://172.16.0.103:2380 clientURLs=https://172.16.0.103:2379 isLeader=false
```

验证etcd集群状态

``` bash
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/etcd/ssl/ca.pem \
          --cert-file=/etc/etcd/ssl/etcd.pem \
          --key-file=/etc/etcd/ssl/etcd-key.pem cluster-health
member 7e218077496bccf9 is healthy: got healthy result from https://172.16.0.101:2379
member 92f1b7c038a4300a is healthy: got healthy result from https://172.16.0.102:2379
member c8611e11b142e510 is healthy: got healthy result from https://172.16.0.103:2379
cluster is healthy //表示安装成功
```

## etcd集群增加节点

将目标节点添加到etcd集群

注意: 需要重新生成新的证书，将新的IP地址添加进去。

``` bash
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/etcd/ssl/ca.pem \
          --cert-file=/etc/etcd/ssl/etcd.pem \
          --key-file=/etc/etcd/ssl/etcd-key.pem member add etcd_node4 https://172.16.0.104:2380
Added member named etcd_etcd4 with ID 5282b16e923af92f to cluster
 
ETCD_NAME="etcd_node4"
ETCD_INITIAL_CLUSTER="etcd_node4=https://172.16.0.104:2380,etcd_node1=https://172.16.0.101:2380,etcd_node2=https://172.16.0.102:2380,etcd_node3=https://172.16.0.103:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

查看成员列表. 此时etcd_node4节点状态为: unstarted 

``` bash
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/etcd/ssl/ca.pem \
          --cert-file=/etc/etcd/ssl/etcd.pem \
          --key-file=/etc/etcd/ssl/etcd-key.pem member list
5282b16e923af92f[unstarted]: peerURLs=https://172.16.0.104:2380
7e218077496bccf9: name=etcd_node1 peerURLs=https://172.16.0.101:2380 clientURLs=https://172.16.0.101:2379 isLeader=true
92f1b7c038a4300a: name=etcd_node2 peerURLs=https://172.16.0.102:2380 clientURLs=https://172.16.0.102:2379 isLeader=false
c8611e11b142e510: name=etcd_node3 peerURLs=https://172.16.0.103:2380 clientURLs=https://172.16.0.103:2379 isLeader=false
```

配置etcd_node4节点的etcd.conf文件

``` bash
# vim /etc/etcd/etcd.conf
ETCD_NAME="etcd_node4" // 节点名称,对应etcd添加节点命令时输出的信息
ETCD_DATA_DIR="/var/lib/etcd/etcd_node4.etcd"
ETCD_LISTEN_PEER_URLS="https://172.16.0.104:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.16.0.104:2379,https://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.0.104:2380"
ETCD_INITIAL_CLUSTER="etcd_node4=https://172.16.0.104:2380,etcd_node1=https://172.16.0.101:2380,etcd_node2=https://172.16.0.101:2380,etcd_node3=https://172.16.0.103:2380" // 集群列表,对应etcd添加节点命令时输出的信息
ETCD_INITIAL_CLUSTER_STATE="existing" // 集群状态,对应etcd添加节点命令时输出的信息
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.0.104:2379"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_PEER_AUTO_TLS="true"
# systemctl enable etcd.service 
# systemctl start etcd.service && systemctl status etcd.service
```

再次查看成员列表. 此时etcd_etcd4节点状态已经显示正常

``` bash
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/etcd/ssl/ca.pem \
          --cert-file=/etc/etcd/ssl/etcd.pem \
          --key-file=/etc/etcd/ssl/etcd-key.pem member list
5282b16e923af92f: name=etcd_node4 peerURLs=https://172.16.0.104:2380 clientURLs=https://172.16.0.104:2379 isLeader=false
7e218077496bccf9: name=etcd_node1 peerURLs=https://172.16.0.101:2380 clientURLs=https://172.16.0.101:2379 isLeader=true
92f1b7c038a4300a: name=etcd_node2 peerURLs=https://172.16.0.102:2380 clientURLs=https://172.16.0.102:2379 isLeader=false
c8611e11b142e510: name=etcd_node3 peerURLs=https://172.16.0.103:2380 clientURLs=https://172.16.0.103:2379 isLeader=false
```

## etcd集群删除节点

删除etcd_etcd4节点

``` bash
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/etcd/ssl/ca.pem \
          --cert-file=/etc/etcd/ssl/etcd.pem \
          --key-file=/etc/etcd/ssl/etcd-key.pem member remove 5282b16e923af92f
Removed member 5282b16e923af92f from cluster
# etcdctl --endpoints=https://127.0.0.1:2379 \
          --ca-file=/etc/etcd/ssl/ca.pem \
          --cert-file=/etc/etcd/ssl/etcd.pem \
          --key-file=/etc/etcd/ssl/etcd-key.pem member list
7e218077496bccf9: name=etcd_node1 peerURLs=https://172.16.0.101:2380 clientURLs=https://172.16.0.101:2379 isLeader=true
92f1b7c038a4300a: name=etcd_node2 peerURLs=https://172.16.0.102:2380 clientURLs=https://172.16.0.102:2379 isLeader=false
c8611e11b142e510: name=etcd_node3 peerURLs=https://172.16.0.103:2380 clientURLs=https://172.16.0.103:2379 isLeader=false
```

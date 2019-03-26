# 部署haproxy + keepalived服务

## Keepalived高可用方案

说明: keepalived软件主要是通过VRRP协议实现高可用功能的，因此还可以作为其他服务的高可用解决方案；下面的解决方案并非完美解决方案，仅供参考学习。

+ `keepalived`在运行过程中周期检查本机的haproxy进程状态，如果检测到haproxy进程异常，则触发重新选主的过程，VIP将飘移到新选出来的主节点，从而实现VIP的高可用。

## Kubernetes高可用方案

`kubernetes`的`Master`节点为三台主机，当前示例的haproxy监听的端口是`8443`，与`kube-apiserver`的端口`6443`不同，避免冲突。

`kubernetes`组件相关组件`kubeclt`、`controller-manager`、`scheduler`、`kube-proxy`等均都通过`VIP`和`haproxy`监听的`8443`端口访问`kube-apiserver`服务。

## 安装haproxy服务
``` bash
# yum install keepalived -y
```

## 配置haproxy服务
``` bash
# cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak_$(date '+%Y%m%d') 
# vim /etc/haproxy/haproxy.cfg
global
    log      127.0.0.1 local3
    maxconn  20480
    chroot   /var/lib/haproxy
    user     haproxy
    group    haproxy
    nbproc   8
    daemon
    quiet
 
defaults
    log         global
    mode        tcp
    option      tcplog
    option      dontlognull
    option      redispatch
    option      forwardfor
    option      http-pretend-keepalive
    retries     3
    redispatch
    contimeout  5000
    clitimeout  50000
    srvtimeout  50000

frontend  kube_https *:8443
    mode               tcp
    maxconn            20480
    default_backend    kube_backend
    
backend  kube_backend
    balance      roundrobin
    server kube-master-01 172.16.0.101:6443 check inter 5000 fall 3 rise 3 weight 1
    server kube-master-02 172.16.0.102:6443 check inter 5000 fall 3 rise 3 weight 1
    server kube-master-03 172.16.0.103:6443 check inter 5000 fall 3 rise 3 weight 1
 
listen haproxy-status
    bind 0.0.0.0:18443
    mode  http
    stats refresh 30s
    stats uri /haproxy-status
    stats realm welcome login\ Haproxy
    stats auth admin:admin
    
# systemctl enable haproxy 
# systemctl restart haproxy
# netstat -ntpl|grep haproxy
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      7456/haproxy        
tcp        0      0 0.0.0.0:18443           0.0.0.0:*               LISTEN      7456/haproxy  
```

## 安装keepalived服务
``` bash
# yum install keepalived -y
```

## 创建健康检查脚本
``` bash
# vim /etc/keepalived/haproxy_check.sh
#!/bin/bash

flag=$(systemctl status haproxy &> /dev/null;echo $?)

if [[ $flag != 0 ]];then
    echo "haproxy is down,close the keepalived"
    systemctl stop keepalived
fi
# chmod +x /etc/keepalived/haproxy_check.sh
```

## 配置keepalived服务
``` bash
# cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak_$(date '+%Y%m%d') 
# cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
 
global_defs {
   notification_email {
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_script haproxy_check {
    script "/etc/keepalived/haproxy_check.sh"
    interval 5
}

vrrp_instance VI_1 {
    state MASTER            // 在备节点设置为BACKUP     
    interface eth0
    virtual_router_id 51
    priority 200            // 备节点的阀值小于主节点
    nopreempt               // MASTER节点故障恢复后不重新抢回VIP
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.16.0.253        // MASTER VIP 
    }
    
    track_script {
        haproxy_check       // 检查脚本
    }
}
 
# systemctl enable keepalived 
# systemctl restart keepalived
# systemctl status keepalived
```



# 部署kube-dns插件

## kube-dns作用

`kube-dns`是`kubernetes`的重要插件，用于完成集群内部`service`的注册和发现。随着`kubernetes`安装和管理体验的进一步完善，`kube-dns`插件势必将成为`kubernetes`默认安装的一部分。

## kube-dns工作原理

kube-dns由`kube-dns`、`dnsmasq-nanny`、`dnsmasq`三个容器构成。

kube-dns服务的核心组件，主要由KubeDNS和SkyDNS组成：

+ KubeDNS负责监听Service和Endpoint的变化情况，并将相关的信息更新到SkyDNS中
+ SkyDNS负责DNS解析，监听在10053端口(tcp/udp)，同时也监听在10055端口提供metrics
+ kube-dns还监听了8081端口，以供健康检查使用

dnsmasq-nanny服务的核心组件：
+ 负责启动dnsmasq，并在配置发生变化时重启dnsmasq
+ dnsmasq的upstream为SkyDNS，即集群内部的DNS解析由SkyDNS负责

sidecar务的核心组件：
+ 负责健康检查和提供DNS metrics(监听在10054端口)

## 下载kube-dns插件

``` bash
# mkdir -p kube-dns && cd kube-dns
# wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-sa.yaml
# wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-cm.yaml
# wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-svc.yaml.base
# wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-controller.yaml.base
# cp kubedns-svc.yaml.base kubedns-svc.yaml && cp kubedns-controller.yaml.base kubedns-controller.yaml
```

## 配置kube-dns服务

需要将__PILLAR__DNS__SERVER__设置为集群环境的IP, 这个IP需要和kubelet的--cluster-dns参数值一致.


``` bash
# diff kubedns-svc.yaml kubedns-svc.yaml.base 
30c30
<   clusterIP: 10.254.0.10
---
>   clusterIP: __PILLAR__DNS__SERVER__
```

## 配置kube-deployment服务

注意：需要将`__PILLAR__DNS__DOMAIN__`设置为集群环境中的域名, 这个`IP`需要和`kubelet`的`--cluster-domain`参数值一致.

大规模集群优化建议: 
+ 增加dns_cpu_limit限制参数: dns_cpu_limit: 200m
+ 修改dns_memory_limit限制参数: dns_memory_limit: 512Mi
+ 修改dns_cpu_requests限制参数: dns_cpu_requests: 100m
+ 修改dns_memory_requests限制参数: dns_memory_requests: 100Mi
+ 修改dnsmasq_cache_size缓存大小: dnsmasq_cache_size=100000
+ 增加dnsmasq_forward_max并发处理数: --dns-forward-max=1000

``` bash
# diff kubedns-controller.yaml kubedns-controller.yaml.base
88c88
<         - --domain=cluster.local.
---
>         - --domain=__PILLAR__DNS__DOMAIN__.
128c128
<         - --server=/cluster.local/127.0.0.1#10053
---
>         - --server=/__PILLAR__DNS__DOMAIN__/127.0.0.1#10053
160,161c160,161
<         - --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local,5,A
<         - --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local,5,A
---
>         - --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,5,A
>         - --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,5,A
```

## 安装kube-dns服务

``` bash
# kubectl apply -f .
configmap "kube-dns" created
deployment "kube-dns" created
serviceaccount "kube-dns" created
service "kube-dns" created
# kubectl get pod -n kube-system
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-3587985970-n1fc3   3/3       Running   0          1h
```
## 验证kube-dns插件功能

检查`my-nginx-svc`的`IP`地址信息

``` bash
# kubectl get svc|grep nginx    
my-nginx-svc           10.254.185.145   <nodes>       80:30080/TCP   9d
```

通过另一个`pod`服务检查是否能够解析到`my-nginx-svc`服务，解析结果显示`10.254.185.145`说明kube-dns服务正常。

``` bash
# kubectl exec my-nginx-1201858851-11vnw -- ping -c 2 my-nginx-svc
PING my-nginx-svc.default.svc.cluster.local (10.254.185.145): 48 data bytes
--- my-nginx-svc.default.svc.cluster.local ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

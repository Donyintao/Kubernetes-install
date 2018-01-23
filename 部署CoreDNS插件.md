# 部署CoreDNS服务

## CoreDNS简介

`CoreDNS`是一个DNS服务器，和Caddy Server具有相同的模型：它链接插件。CoreDNS是云本土计算基金会启动阶段项目。

`CoreDNS`是`SkyDNS`的继任者；`SkyDNS`是一个薄层，暴露了DNS中的etcd中的服务。`CoreDNS`建立在这个想法上，是一个通用的DNS服务器，可以与多个后端(etcd，kubernetes等)进行通信。

`CoreDNS`旨在成为一个快速灵活的DNS服务器。这里的关键灵活指的是：使用`CoreDNS`，您可以使用DNS数据进行所需的操作；还可以自已写插件来实现DNS的功能。

CoreDNS可以通过UDP/TCP(旧式的DNS)，TLS(RFC 7858)和gRPC(不是标准)监听DNS请求。

## 下载CoreDNS镜像文件

``` bash
# mkdir -p coredns && coredns
# wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed
# cp coredns.yaml.sed coredns.yaml
```

## 配置CoreDNS服务

注意：需要将`CLUSTER_DOMAIN`设置为集群环境中的域名, 这个域名需要和`kubelet`的`--cluster-domain`参数值一致.

注意：需要将`REVERSE_CIDRS`设置为集群环境中SVC地址段，这个地址段需要和`kube-apiserver`的`--service-cluster-ip-range`参数值一致.

注意：需要将`CLUSTER_DNS_IP`设置为集群环境的IP, 这个`IP`需要和`kubelet`的`--cluster-dns`参数值一致.

``` bash
# diff coredns.yaml coredns.yaml.sed 
52c52
<         kubernetes cluster.local 172.21.0.0/16
---
>         kubernetes CLUSTER_DOMAIN REVERSE_CIDRS
147c147
<   clusterIP: 172.21.0.2
---
>   clusterIP: CLUSTER_DNS_IP
```
## 安装CoreDNS服务

``` bash
# kubectl apply -f coredns.yaml 
serviceaccount "coredns" created
clusterrole "system:coredns" created
clusterrolebinding "system:coredns" created
configmap "coredns" created
deployment "coredns" created
service "kube-dns" created
# kubectl get pod -n kube-system
NAME                       READY     STATUS    RESTARTS   AGE
coredns-5984fb8cbb-4ss5v   1/1       Running   0          19s
coredns-5984fb8cbb-79wnz   1/1       Running   0          20s
```
## 验证CoreDNS功能

待补充...

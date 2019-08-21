# 配置kubeconfig访问多个集群
本篇文章主要讲如何使用一个配置文件，在将集群、用户和上下文定义在一个或多个配置文件中之后，用户可以使用`kubectl config use-context`命令快速的在不同集群之间进行切换。

## 定义集群、用户和上下文
假设当前环境有两个集群，一个是生产环境，一个是测试环境。
``` bash
# mkdir -p ~/.kube && cd ~/.kube
# vim config
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem   # dev集群CA证书
    server: https://10.8.0.136:6443                     # dev集群APIServer地址	
  name: kubernetes-cluser-dev                           # dev集群名称(自定义)
- cluster:            
    certificate-authority: /etc/kubernetes/ssl/ca.pem   # pro集群CA证书	
    server: https://10.8.1.136:6443                     # pro集群CA证书
  name: kubernetes-cluser-pro                           # pro集群名称(自定义)
contexts:
- context:                                              # 定义集群上下文
    cluster: kubernetes-cluser-dev
    user: cluser-dev-admin
  name: kubernetes-cluser-dev                           # dev集群context名称(自定义)
- context:
    cluster: kubernetes-cluser-pro				
    user: cluser-pro-admin						
  name: kubernetes-cluser-pro                           # pro集群context名称(自定义)	
current-context: kubernetes-cluser-dev                  # 定义默认集群
kind: Config
preferences: {}
users:
- name: cluser-dev-admin                                # dev集群用户名称(自定义)
  user:
    client-certificate: /etc/kubernetes/ssl/kubelet.pem # dev集群客户证书
    client-key: /etc/kubernetes/ssl/kubelet-key.pem		
- name: cluser-pro-admin                                # pro集群用户名称(自定义)
  user:
    client-certificate: /etc/kubernetes/ssl/kubelet.pem # pro集群客户证书
    client-key: /etc/kubernetes/ssl/kubelet-key.pem
```

* 查看所有的可使用的kubernetes集群角色

``` bash
# kubectl config get-contexts
CURRENT   NAME                    CLUSTER                 AUTHINFO           NAMESPACE
*         kubernetes-cluser-dev   kubernetes-cluser-dev   cluser-dev-admin   
          kubernetes-cluser-pro   kubernetes-cluser-pro   cluser-pro-admin 
```

* 切换kubernetes配置

``` bash
# kubectl config use-context kubernetes-cluser-pro
Switched to context "kubernetes-cluser-pro".
# kubectl config get-contexts                     
CURRENT   NAME                    CLUSTER                 AUTHINFO           NAMESPACE
          kubernetes-cluser-dev   kubernetes-cluser-dev   cluser-dev-admin   
*         kubernetes-cluser-pro   kubernetes-cluser-pro   cluser-pro-admin
```
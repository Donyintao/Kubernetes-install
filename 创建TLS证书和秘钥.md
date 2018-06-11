# 创建TLS证书和秘钥

`kubernetes`系统各组件需要使用`TLS`证书对通信进行加密，本文档使用`CloudFlare`的PKI工具集[cfssl](https://github.com/cloudflare/cfssl)来生成Certificate Authority(CA)证书和秘钥文件，CA 是自签名的证书，用来签名后续创建的其它TLS证书。

## 安装 `CFSSL`

``` bash
# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/local/bin/cfssl
# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/local/bin/cfssljson
# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/local/bin/cfssl-certinfo
# chmod +x /usr/local/bin/cfssl*
```

## 创建CA(Certificate Authority)

创建CA配置文件

``` bash
# cfssl print-defaults config > ca-config.json
# cat ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```

创建CA证书签名请求

``` bash
# cfssl print-defaults csr > ca-csr.json 
# cat ca-csr.json 
{
  "CN": "kubernetes",
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
# cfssl gencert -initca ca-csr.json | cfssljson -bare ca

```

## 创建kube-apiserver证书

创建kube-apiserver证书签名请求

注意：默认kube-apiserver证书没有权限访问API接口, 会提示: Unauthorized

注意：如果kube-apiserver证书访问API接口, 需要设置: ["O": "system:masters"]

注意：此处需要将dns ip(首个IP地址)、etcd、k8s-master节点的ip全部加上.

``` bash
# cfssl print-defaults csr > kube-apiserver-csr.json
# cat kube-apiserver-csr.json 
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "172.21.0.1",
      "172.16.30.171",
      "172.16.30.172",
      "172.16.30.173",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
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
``` 

生成kubernetes证书和私钥

``` bash
# cfssl gencert -ca=ca.pem \
                -ca-key=ca-key.pem \
                -config=ca-config.json \
                -profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver
```

## 创建kube-controller-manager证书

创建kube-controller-manager证书签名请求

``` bash
# cfssl print-defaults csr > kube-controller-manager-csr.json 
# cat kube-controller-manager-csr.json 
{
  "CN": "system:kube-controller-manager",
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
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
``` 
生成kube-controller-manager证书和私钥

``` bash
# cfssl gencert -ca=ca.pem \
                -ca-key=ca-key.pem \
                -config=ca-config.json \
                -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

## 创建kube-scheduler证书

创建kube-scheduler证书签名请求

``` bash
# cfssl print-defaults csr > kube-scheduler-csr.json 
# cat kube-scheduler-csr.json 
{
  "CN": "system:kube-scheduler",
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
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
```
 
生成kube-scheduler证书和私钥

``` bash
# cfssl gencert -ca=ca.pem \
                -ca-key=ca-key.pem \
                -config=ca-config.json \
                -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

## 创建kubelet证书

创建kubelet证书签名请求

``` bash
# cfssl print-defaults csr > kubelet-csr.json
# cat kubelet-csr.json
{
  "CN": "kubelet",
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
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```
 
生成kubelet证书和私钥

``` bash
# cfssl gencert -ca=ca.pem \
                -ca-key=ca-key.pem \
                -config=ca-config.json \
                -profile=kubernetes kubelet-csr.json | cfssljson -bare kubelet
```

## 创建kube-proxy证书

创建kube-proxy证书签名请求

``` bash
# cfssl print-defaults csr > kube-proxy-csr.json
# cat kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
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
      "O": "system:node-proxier",
      "OU": "System"
    }
  ]
}
```
 
生成kube-proxy客户端证书和私钥

``` bash
# cfssl gencert -ca=ca.pem \
                -ca-key=ca-key.pem \
                -config=ca-config.json \
                -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

## 证书校验

校验kube-apiserver证书

``` bash
# cfssl-certinfo -cert kube-apiserver.pem
{
  "subject": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "system:masters",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "system:masters",
      "System",
      "kubernetes"
    ]
  },
  "issuer": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "serial_number": "533666226632105718421042600083075622217402341392",
  "sans": [
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local",
    "127.0.0.1",
    "172.21.0.1",
    "172.16.30.171",
    "172.16.30.172",
    "172.16.30.173"
  ],
  "not_before": "2017-07-31T08:57:00Z",
  "not_after": "2018-07-31T08:57:00Z",
  "sigalg": "SHA256WithRSA",
  "authority_key_id": "6B:68:CF:57:62:6B:60:7E:F3:2C:AC:1A:20:6F:27:6A:EA:84:98:A8",
  "subject_key_id": "3C:6C:67:14:69:F8:42:2A:5C:3C:28:65:B6:A3:95:80:49:A6:6:C",
  "pem": "-----BEGIN CERTIFICATE-----
MIIEkDCCA3igAwIBAgIUEdNzDqRQMswGL4KikzjnizkfBS4wDQYJKoZIhvcNAQEL
BQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0Jl
aUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwpr
dWJlcm5ldGVzMB4XDTE3MDcyNzA5MjcwMFoXDTE4MDcyNzA5MjcwMFowcDELMAkG
A1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxFzAV
BgNVBAoTDnN5c3RlbTptYXN0ZXJzMQ8wDQYDVQQLEwZTeXN0ZW0xEzARBgNVBAMT
Cmt1YmVybmV0ZXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQClXFE1
qVQ9HiHDEbyfDMqsrO8p0Rn02ta+xWAbmJhwgNstFfuW0Lz9XmtpclDRfF2U5QOJ
X7TrTZz2xhjRxXzUb/4EU035VH273tb+3+orUbggMcUzavpbm0zFqqSeSTxIoWhw
wiIUG33BR7i6kvyH7eHraq/vYn8NbG2t8ufoJFgPys6zjC9rDWqNlBXume69n8BD
HTfDQUgUVLZDDZyef+KwvtziHUtEgEakaI9MgDV3CdkMAvXrnIeiMHQzRBen3gli
zk4i+OCWd9oI7cB7oqvXUm+pTEAzOPQaGkkq7A2R8UHTFgOyAkw8saKwRvBacWhm
BDa/+CVYKfiNBzDRAgMBAAGjggErMIIBJzAOBgNVHQ8BAf8EBAMCBaAwHQYDVR0l
BBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMAwGA1UdEwEB/wQCMAAwHQYDVR0OBBYE
FHfVB5vi0gEh2rGjBzWVr9+2Jrs9MB8GA1UdIwQYMBaAFGHhP32/2ThF4VlOuaKj
iKbG/CMcMIGnBgNVHREEgZ8wgZyCCmt1YmVybmV0ZXOCEmt1YmVybmV0ZXMuZGVm
YXVsdIIWa3ViZXJuZXRlcy5kZWZhdWx0LnN2Y4Iea3ViZXJuZXRlcy5kZWZhdWx0
LnN2Yy5jbHVzdGVygiRrdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9j
YWyHBH8AAAGHBAr+AAGHBMCoA1+HBMCoA2CHBMCoA2MwDQYJKoZIhvcNAQELBQAD
ggEBABlTNX+MVTfViozPrwH6QkXfbHTH9kpsm9SPZhpzjON4pAcY5kP3t6DInX9D
SdivyuVn3jJz6BaBIoUh5RJRsq6ArMpbl1g7dyZnHZXPjLtMAFYGgnBjH6XVEQ1f
FZbSZjvbti/l7SH7f9aqtywzqNCDqmwx+2gNoWwd11y0A7zxMVK28l6apbMfcVHL
rHLKikoV+sLmvKCLdh7/qrTToono0j5nMzuQWfNU3UsNHOZZ1uNUQsuurv95LUWG
5t3PKpRoi0Z5kePBdLoD1CHqS1DEPkZt+sj6e6vqQSBAM8usNEUwi7ASOY2zAaMG
aDz1i4/WZhJSUQyDfx7HzJpAmBE=
-----END CERTIFICATE-----"
}
```
## 分发证书

将`TLS`证书拷贝到`Kubernetes Master`和`Kubernetes node`的配置目录

``` bash
# mkdir -p /etc/kubernetes/ssl && cp /tmp/ssl/*.pem /etc/kubernetes/ssl
```

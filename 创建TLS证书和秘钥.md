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

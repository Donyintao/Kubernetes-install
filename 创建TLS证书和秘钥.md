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

``` bash
# cfssl print-defaults csr > kubernetes-csr.json
# cat kube-apiserver-csr.json 
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "172.16.0.1",
      "192.168.100.110",
      "192.168.100.111",
      "192.168.100.112",
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
# cfssl print-defaults csr > admin-csr.json 
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
# cat > kubelet-csr.json << EOF
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
EOF
```
 
生成kubelet证书和私钥

``` bash
cfssl gencert -ca=ca.pem \
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
    "10.10.0.1",
    "192.168.100.110",
    "192.168.100.111",
    "192.168.100.112"
  ],
  "not_before": "2017-07-31T08:57:00Z",
  "not_after": "2018-07-31T08:57:00Z",
  "sigalg": "SHA256WithRSA",
  "authority_key_id": "6B:68:CF:57:62:6B:60:7E:F3:2C:AC:1A:20:6F:27:6A:EA:84:98:A8",
  "subject_key_id": "3C:6C:67:14:69:F8:42:2A:5C:3C:28:65:B6:A3:95:80:49:A6:6:C",
  "pem": "-----BEGIN CERTIFICATE-----\nMIIEkDCCA3igAwIBAgIUXXpr1pOjvLUxQVv+JMKjwgvQ2BAwDQYJKoZIhvcNAQEL\nBQAwZTELMAkGA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0Jl\naUppbmcxDDAKBgNVBAoTA2s4czEPMA0GA1UECxMGU3lzdGVtMRMwEQYDVQQDEwpr\ndWJlcm5ldGVzMB4XDTE3MDczMTA4NTcwMFoXDTE4MDczMTA4NTcwMFowcDELMAkG\nA1UEBhMCQ04xEDAOBgNVBAgTB0JlaUppbmcxEDAOBgNVBAcTB0JlaUppbmcxFzAV\nBgNVBAoTDnN5c3RlbTptYXN0ZXJzMQ8wDQYDVQQLEwZTeXN0ZW0xEzARBgNVBAMT\nCmt1YmVybmV0ZXMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDIxzDb\nQP5zp8k8ydDrZPfV8KDkWWDnFvNhE2R0XUeD8d3A/MCjqTZh+ugtDZanzWx4HoYb\nTEnYJZbpKnVb99gQ+laIHLOs6pwl+ADC7k6DStUv4wSBZkHzHTMxjmAxdwemyVEL\nAJfZonchEIb9ouMwLTVSLjjr63DVbg0cRDaEQ+PQFcPenMCzisQniytut6z8wJX0\nbB6Qsb8RrVLusIUy/GjwWor11GV0FrScujKDnH37rN0Xj5cMe3Zd0jj4Jv641fLs\nkIpipXSXFkFTSB2ApdOT61bO4A1qoQlxni8/nJqVri4NKW6AAsq4cAisxYD7N/uU\n2ih2+FIkKqohpXe1AgMBAAGjggErMIIBJzAOBgNVHQ8BAf8EBAMCBaAwHQYDVR0l\nBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMAwGA1UdEwEB/wQCMAAwHQYDVR0OBBYE\nFDxsZxRp+EIqXDwoZbajlYBJpgYMMB8GA1UdIwQYMBaAFGtoz1dia2B+8yysGiBv\nJ2rqhJioMIGnBgNVHREEgZ8wgZyCCmt1YmVybmV0ZXOCEmt1YmVybmV0ZXMuZGVm\nYXVsdIIWa3ViZXJuZXRlcy5kZWZhdWx0LnN2Y4Iea3ViZXJuZXRlcy5kZWZhdWx0\nLnN2Yy5jbHVzdGVygiRrdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9j\nYWyHBH8AAAGHBAoKAAGHBMCoZG6HBMCoZG+HBMCoZHAwDQYJKoZIhvcNAQELBQAD\nggEBADNlsPPPhcx3HpjztYmE7vtH6d+8kB8bhML+fWMD17xOnE1xM5mi62tcP8vf\nbQ9v6Q4L6EKXyruvkkSiQsdoQLF5rj3PBqF1vxw8StLY04YSP1Jn11ftl9akAbvh\nUJPXTzIRPfqzkrvQwwZS3clYly3mQNgEv60Rrnc1gvRxyWFu0lOpbldoZUamYOYJ\nV2w+dPmLM8kdy5pIg5dndNBUi9oSqCOpCMaFeJgKLmSmTWHLhzUoXwOvSrrBsaK4\n/57/fXF5bkTaBwwG7O2QAvzwJFKzGsjkQiAcgZCy7FhRgprQYeg6gTIn5RvpmydC\nkaZmIrJkdAN7RXJZ4fbUxu+whkc=\n-----END CERTIFICATE-----\n"
}
```

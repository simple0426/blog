---
title: 证书生成工具cfssl
tags:
  - cfssl
categories:
  - skill
date: 2020-07-15 23:01:52
---


# 下载cfssl工具

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl*
mv cfssl_linux-amd64 /usr/bin/cfssl
mv cfssljson_linux-amd64 /usr/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

# 创建CA

## 初始化配置

```
# config
cfssl print-defaults config > ca-config.json
# csr
cfssl print-defaults csr > ca-csr.json
```

## CA配置文件-范例

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
```

* ca-config.json：可以定义多个profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时需要使用某个profile
* signing：表示该证书可用于签名其他证书
* server auth：表示client可以用该CA对server提供的证书进行验证
* client auth：表示server可以用该CA对client提供的证书进行验证

## CA证书签名请求-范例


```
cat > ca-csr.json <<EOF
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
            "L": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

* CN-Common Name：kube-apiserver从证书中提取该字段作为请求的用户名(User Name)；浏览器使用该字段验证网站是否合法
* C-countryName：国家名
* ST-stateOrProvinceName：省份
* L-localityName：城市名
* O-Organization：组织机构名，kube-apiserver从证书中提取该字段作为请求用户所属的组(Group)
* OU-OrganizationUnitName：组织下的部门名称
* hosts：如果hosts字段不为空，则需要指定使用该证书的IP或域名列表

## 生成CA证书和私钥

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
ls ca*.pem
# 证书信息查看
cfssl-certinfo -cert ca.pem
```

# CA签发普通证书

```
cat > nginx.myapp.com-csr.json <<EOF
{
  "CN": "nginx.myapp.com",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes nginx.myapp.com-csr.json | cfssljson -bare nginx.myapp.com
```


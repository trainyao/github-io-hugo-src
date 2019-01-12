---
author: "trainyao"
date: "2019-01-12"
title: "使用 kubernetes 的 CA 证书自签名服务端 & 客户端证书"
linktitle: "使用 kubernetes 的 CA 证书自签名服务端 & 客户端证书"
weight: 10
---


1. 安装 `cfssl`

```shell
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

export PATH=/usr/local/bin:$PATH
```

2. 找到 kubernetes 的根证书 `ca.pem`, `ca-key.pem`, `ca-config.json`

3. 生成证书请求配置文件, 可以替换 `usage` 为其他名字, 替换 `usage.common.name` 为服务器域名

```shell
cat > usage.json <<EOF
{
  "CN": "usage.common.name",
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
  ],
    "ca": {
       "expiry": "87600h"
    }
}
EOF
```

4. 用 `cfssl` 工具签名服务端证书

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes usage.json | cfssljson -bare usage
$ ls usage*
usage.csr  usage.json  usage-key.pem  usage.pem
```

5. 使用 `openssl` 签名客户端证书

```shell
openssl -in usage.pem -out usage.client.pem
```

6. 使用 `usage.pem` `usage-key.pem` `usage.client.pem` 愉快的玩耍

参考资料:

- [创建TLS证书和秘钥](https://jimmysong.io/kubernetes-handbook/practice/create-tls-and-secret-key.html)
- [Admission control webhook with self-signed CA in "caBundle" is not trusted #61171](https://github.com/kubernetes/kubernetes/issues/61171)

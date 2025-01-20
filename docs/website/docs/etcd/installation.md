# ETCD 安装

## ETCD 证书配置

基于 [环境准备 / PKI 配置](/prepare/pki) 中生成的 ca 证书，签发 ETCD 证书

```bash
# ca 证书
# -ca=/etc/pki/ca/ca.pem
# -ca-key=/etc/pki/ca/ca-key.pem
# -config=/etc/pki/ca/ca-config.json
# -profile=client
```

### ETCD 证书规划

签发3种证书：

`admin` 用于 etcdctl
`member` 用于 etcd client 与 etcd peer
`node` 用于第三方调用

创建 ETCD 证书路径

```bash
mkdir -p /root/apisix_compose/cert/ca
cp /etc/pki/ca/{ca.pem,ca-key.pem,ca-config.json} /root/apisix_compose/cert/ca/
mkdir -p /root/apisix_compose/cert/etcd
```

### 创建证书签名请求

```bash
cat <<EOF | sudo tee /root/apisix_compose/cert/etcd/admin-etcd-csr.json
{
  "CN": "etcd.kubegg.local",
  "hosts": [
    "kubegg",
    "etcd.kubegg",
    "etcd.kubegg.io",
    "kubegg.local",
    "etcd.kubegg.local",
    "127.0.0.1",
    "10.1.1.11",
    "localhost"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hubei",
      "L": "Wuhan",
      "O": "Kubegg",
      "OU": "www"
    }
  ]
}

EOF
```

```bash
cat <<EOF | sudo tee /root/apisix_compose/cert/etcd/member-etcd-csr.json
{
  "CN": "etcd.kubegg.local",
  "hosts": [
    "kubegg",
    "etcd.kubegg",
    "etcd.kubegg.io",
    "kubegg.local",
    "etcd.kubegg.local",
    "etcd-1",
    "etcd-2",
    "etcd-3",
    "127.0.0.1",
    "10.1.1.11",
    "localhost"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hubei",
      "L": "Wuhan",
      "O": "Kubegg",
      "OU": "www"
    }
  ]
}

EOF
```

```bash
cat <<EOF | sudo tee /root/apisix_compose/cert/etcd/apisix-etcd-csr.json
{
  "CN": "etcd.kubegg.local",
  "hosts": [
    "kubegg",
    "etcd.kubegg",
    "etcd.kubegg.io",
    "kubegg.local",
    "etcd.kubegg.local",
    "apisix",
    "apisix.kubegg",
    "apisix.kubegg.io",
    "127.0.0.1",
    "10.1.1.11",
    "localhost"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hubei",
      "L": "Wuhan",
      "O": "Kubegg",
      "OU": "www"
    }
  ]
}

EOF
```

### 生成证书与私钥

```bash
# admin
cfssl gencert -ca=/root/apisix_compose/cert/ca/ca.pem -ca-key=/root/apisix_compose/cert/ca/ca-key.pem -config=/root/apisix_compose/cert/ca/ca-config.json -profile=client /root/apisix_compose/cert/etcd/admin-etcd-csr.json | cfssljson -bare /root/apisix_compose/cert/etcd/admin-etcd

# member
cfssl gencert -ca=/root/apisix_compose/cert/ca/ca.pem -ca-key=/root/apisix_compose/cert/ca/ca-key.pem -config=/root/apisix_compose/cert/ca/ca-config.json -profile=peer /root/apisix_compose/cert/etcd/member-etcd-csr.json | cfssljson -bare /root/apisix_compose/cert/etcd/member-etcd

# apisix
cfssl gencert -ca=/root/apisix_compose/cert/ca/ca.pem -ca-key=/root/apisix_compose/cert/ca/ca-key.pem -config=/root/apisix_compose/cert/ca/ca-config.json -profile=client /root/apisix_compose/cert/etcd/apisix-etcd-csr.json | cfssljson -bare /root/apisix_compose/cert/etcd/apisix-etcd
```

## ETCD 容器安装部署

## ETCD 健康检查

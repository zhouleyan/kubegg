# PKI 配置

## 安装 cfssl

下载地址：[https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64](https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64)

下载地址：[https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64](https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64)

下载地址：[https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl-certinfo_1.6.5_linux_amd64](https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl-certinfo_1.6.5_linux_amd64)

下载 `cfssl`、`cfssljson`、`cfssl-certinfo`

```bash
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64 -O /usr/local/bin/cfssl && \
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64 -O /usr/local/bin/cfssljson && \
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl-certinfo_1.6.5_linux_amd64 -O /usr/local/bin/cfssl-certinfo && \
chmod +x /usr/local/bin/{cfssl,cfssljson,cfssl-certinfo}
```

## CA 证书生成

### 准备 CA 配置文件与签名请求

cfssl `config`、`csr` 默认配置

```bash
mkdir -p /etc/pki/ca
# cfssl print-defaults config > /etc/pki/ca/ca-config.json
# cfssl print-defaults csr > /etc/pki/ca/ca-csr.json
```

生成 CA 配置文件 `ca-config.json`

```bash
cat <<EOF | sudo tee /etc/pki/ca/ca-config.json
{
  "signing": {
    "default": {
      "expiry": "438000h"
    },
    "profiles": {
      "peer": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "438000h"
      },
      "client": {
        "usages": [
            "signing",
            "key encipherment",
            "client auth"
        ],
        "expiry": "438000h"
      },
      "server": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth"
        ],
        "expiry": "438000h"
      }
    }
  }
}

EOF
```

`signing` 表示该证书可用于签名其它证书，生成的 ca.pem 证书中 CA=TRUE
`server auth` 表示可以用该 CA 对 server 提供的证书进行验证
`client auth` 表示可以用该 CA 对 client 提供的证书进行验证
`profile peer` 包含了 server auth 和 client auth，所以可以签发三种不同类型证书
`profile client` 在后面客户端 kubeconfig 证书管理中用到
`expiry` 证书有效期，默认 50 年

生成 CA 证书签名请求 `ca-csr.json`

```bash
cat <<EOF | sudo tee /etc/pki/ca/ca-csr.json
{
  "CN": "kubegg.io",
  "key": {
    "algo": "ed25519"
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hubei",
      "L": "Wuhan",
      "O": "kubegg",
      "OU": "kubegg"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}

EOF
```

### 生成 CA 证书与私钥

```bash
cfssl gencert -initca /etc/pki/ca/ca-csr.json | cfssljson -bare /etc/pki/ca/ca

# 生成 ca.csr、ca.pem、ca-key.pem
# 校验证书
# openssl x509 -noout -text -in /etc/pki/ca/ca.pem
# cfssl-certinfo -cert /etc/pki/ca/ca-key.pem
```

## 签发 infra 证书

用于 client 连接

### 准备证书签名请求

```bash
mkdir -p /etc/pki/infra
```

```bash
cat <<EOF | sudo tee /etc/pki/infra/infra-csr.json
{
  "CN": "infra.kubegg.io",
  "hosts": [],
  "key": {
    "algo": "ed25519"
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hubei",
      "L": "Wuhan",
      "O": "kubegg",
      "OU": "infra"
    }
  ]
}

EOF
```

### 生成证书与私钥

```bash
cfssl gencert -ca=/etc/pki/ca/ca.pem -ca-key=/etc/pki/ca/ca-key.pem -config=/etc/pki/ca/ca-config.json -profile=client /etc/pki/infra/infra-csr.json | cfssljson -bare /etc/pki/infra/infra
```

## 分发证书

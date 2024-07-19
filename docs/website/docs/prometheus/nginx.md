# Nginx 安装

## Nginx 证书配置

### 签发 Nginx 证书

基于 [环境准备 / PKI 配置](/prepare/pki) 中生成的 ca 证书，签发 Nginx 证书

```bash
# ca 证书
# -ca=/etc/pki/ca/ca.pem
# -ca-key=/etc/pki/ca/ca-key.pem
# -config=/etc/pki/ca/ca-config.json
# -profile=client
```

#### 创建 Nginx 证书请求

```bash
cat <<EOF | sudo tee /etc/nginx/conf.d/cert/kubegg-csr.json
{
  "CN": "h.kubegg.local",
  "hosts": [
    "kubegg",
    "h.kubegg",
    "p.kubegg",
    "g.kubegg",
    "a.kubegg",
    "kubegg.io",
    "h.kubegg.io",
    "p.kubegg.io",
    "g.kubegg.io",
    "a.kubegg.io",
    "kubegg.local",
    "h.kubegg.local",
    "p.kubegg.local",
    "g.kubegg.local",
    "a.kubegg.local",
    "127.0.0.1",
    "10.1.1.11",
    "localhost"
  ],
  "key": {
    "algo": "ecdsa",
    "size": 256
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

#### 生成证书与私钥

```bash
cfssl gencert -ca=/etc/pki/ca/ca.pem -ca-key=/etc/pki/ca/ca-key.pem -config=/etc/pki/ca/ca-config.json -profile=server /etc/nginx/conf.d/cert/kubegg-csr.json | cfssljson -bare /etc/nginx/conf.d/cert/kubegg
```

`/etc/nginx/conf.d/cert/kubegg.pem` Nginx Server 证书文件
`/etc/nginx/conf.d/cert/kubegg-key.pem` Nginx Server 私钥

查看证书

```bash
openssl x509 -noout -text -in /etc/nginx/conf.d/cert/kubegg.pem

# curl --cacert cacertfile --cert certfile --key keyfile https://example.com
```

## Nginx Server 配置

### 创建 Nginx 配置文件夹

```bash
mkdir -p /etc/nginx/conf.d/cert && \
mkdir -p /etc/nginx/conf.d/haproxy && \
rm -rf /etc/nginx/conf.d/default.conf
```

### 创建 Nginx 主配置

```bash
cat <<EOF | tee /etc/nginx/nginx.conf
user  nginx;
worker_processes  1;

error_log  /var/log/nginx-error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '\$remote_addr - \$remote_user [\$time_local]"\$request" '
                      '\$status \$body_bytes_sent"\$http_referer" '
                      '"\$http_user_agent" "\$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile on;
    gzip  on;
    keepalive_timeout  65;
    include /etc/nginx/conf.d/*.conf;
}
EOF
```

### Home 服务配置

用于 nginx_exporter 监控探针接入

```bash
cat <<EOF | tee /etc/nginx/conf.d/home.conf
# DEFAULT SERVER @ 80 / 443

# include haproxy admin webui upstream definition
include /etc/nginx/conf.d/haproxy/upstream-*.conf;

server {
    listen                    80;
    server_name               .kubegg .kubegg.io .kubegg.local localhost;
    return                    301 https://\$host\$request_uri;
}

server {
    server_name               h.kubegg h.kubegg.local localhost;

    listen                    443 ssl default_server;
    ssl_certificate           /etc/nginx/conf.d/cert/kubegg.pem;
    ssl_certificate_key       /etc/nginx/conf.d/cert/kubegg-key.pem;
    ssl_session_timeout       5m;
    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers               ECDHE-ECDSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;

    # home server
    location / {
        root        /www;
        index       index.html;
        autoindex   on;
        autoindex_exact_size on;
        autoindex_localtime on;
        autoindex_format html;
    }

    # liveness probe
    location /nginx {
        stub_status on;
        access_log off;
    }

    # proxy pass haproxy admin webui requests
    include /etc/nginx/conf.d/haproxy/location-*.conf;
}

EOF
```

### Grafana 服务配置

```bash
cat <<EOF | tee /etc/nginx/conf.d/grafana.conf
# INFRA PORTAL grafana: g.kubegg.local -> 127.0.0.1:3000

upstream grafana {
    server 127.0.0.1:3000 max_fails=0;
}

server {
    server_name               g.kubegg g.kubegg.io g.kubegg.local;
    listen                    443;
    ssl_certificate           /etc/nginx/conf.d/cert/kubegg.pem;
    ssl_certificate_key       /etc/nginx/conf.d/cert/kubegg-key.pem;
    ssl_session_timeout       5m;
    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers               ECDHE-ECDSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;

    access_log  /var/log/nginx/grafana.log;
    location / {
        proxy_pass http://grafana/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-Scheme \$scheme;
        proxy_set_header X-Real_IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_connect_timeout 5;
        proxy_read_timeout 120s;
        proxy_next_upstream error;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";

    }
}

EOF
```

### Prometheus 服务配置

```bash
cat <<EOF | tee /etc/nginx/conf.d/prometheus.conf
# INFRA PORTAL prometheus: p.kubegg.local -> 127.0.0.1:9090

upstream prometheus {
    server 127.0.0.1:9090 max_fails=0;
}

server {
    server_name               p.kubegg p.kubegg.io p.kubegg.local;
    listen                    443;
    ssl_certificate           /etc/nginx/conf.d/cert/kubegg.pem;
    ssl_certificate_key       /etc/nginx/conf.d/cert/kubegg-key.pem;
    ssl_session_timeout       5m;
    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers               ECDHE-ECDSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;

    access_log  /var/log/nginx/prometheus.log;
    location / {
        proxy_pass http://prometheus/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-Scheme \$scheme;
        proxy_set_header X-Real_IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_connect_timeout 5;
        proxy_read_timeout 120s;
        proxy_next_upstream error;
    }
}

EOF
```

### Alertmanager 服务配置

```bash
cat <<EOF | tee /etc/nginx/conf.d/alertmanager.conf
# INFRA PORTAL alertmanager: a.kubegg.local -> 127.0.0.1:9093

upstream alertmanager {
    server 127.0.0.1:9093 max_fails=0;
}

server {
    server_name               a.kubegg a.kubegg.io a.kubegg.local;
    listen                    443;
    ssl_certificate           /etc/nginx/conf.d/cert/kubegg.pem;
    ssl_certificate_key       /etc/nginx/conf.d/cert/kubegg-key.pem;
    ssl_session_timeout       5m;
    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers               ECDHE-ECDSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;

    access_log  /var/log/nginx/alertmanager.log;
    location / {
        proxy_pass http://alertmanager/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-Scheme \$scheme;
        proxy_set_header X-Real_IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_connect_timeout 5;
        proxy_read_timeout 120s;
        proxy_next_upstream error;
    }
}

EOF
```

## 创建 Nginx 静态文件

```bash
# 创建 Nginx Home 文件夹
mkdir -p /www
```

```html
# 创建 Index 静态文件
cat <<EOF | tee /www/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Kubegg</title>
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
    <meta name="description" content="Kubegg Home Page">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0">
</head>
<body data-page="index.html">
    Kubegg Home
</body>
</html>
EOF
```

## Nginx 服务启动

```bash
systemctl enable nginx
systemctl restart nginx
```

## Nginx exporter 部署配置

### Nginx exporter 配置

```bash
cat <<EOF | tee /etc/default/nginx_exporter
NGINX_EXPORTER_OPTS="-nginx.scrape-uri http://127.0.0.1:80/nginx"
EOF
```

### Nginx exporter 启动

```bash
systemctl enable nginx_exporter
systemctl restart nginx_exporter
```

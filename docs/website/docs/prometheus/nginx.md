# Nginx 安装

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

### 创建 Nginx 服务配置

```bash
# infra_portal:                     # infra services exposed via portal
#       home         : {domain: h.pigsty}
#       grafana      : {domain: g.pigsty ,endpoint: "${admin_ip}:3000" ,websocket: true }
#       prometheus   : {domain: p.pigsty ,endpoint: "${admin_ip}:9090" }
#       alertmanager : {domain: a.pigsty ,endpoint: "${admin_ip}:9093" }
#       blackbox     : {endpoint: "${admin_ip}:9115" }
#       loki         : {endpoint: "${admin_ip}:3100" }
```

```bash
cat <<EOF | tee /etc/nginx/conf.d/grafana.conf
# INFRA PORTAL grafana: g.kubegg.local -> 127.0.0.1:3000

upstream grafana {
    server 127.0.0.1:3000 max_fails=0;
}

server {
    server_name g.kubegg.local;
    listen      80;

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

## Nginx 证书配置

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

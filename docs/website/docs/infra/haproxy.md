# HAProxy 安装

## Linux 主机准备

基于上一步安装完 Infra 的主机，继续安装 HAProxy

## HAProxy 配置

### 创建 HAProxy 文件夹

```bash
mkdir -p /etc/haproxy && \
chmod 0700 /etc/haproxy
```

### HAProxy environment file

```bash
touch /etc/default/haproxy && \
chmod 0644 /etc/default/haproxy
```

### HAProxy systemd service

```bash
cat <<EOF | tee /usr/lib/systemd/system/haproxy.service
[Unit]
Description=HAProxy2 Load Balancer
Documentation=https://www.haproxy.org/download/2.0/doc/configuration.txt
After=network.target

[Service]
LimitNOFILE=16777216
LimitNPROC=infinity
LimitCORE=infinity
Environment="CONFIG=/etc/haproxy/"
EnvironmentFile=-/etc/default/haproxy
ExecStartPre=/usr/sbin/haproxy -f \$CONFIG -c -q
ExecStart=/usr/sbin/haproxy -Ws -f \$CONFIG \$OPTIONS
ExecReload=/usr/sbin/haproxy -f \$CONFIG -c -q
ExecReload=/bin/kill -USR2 \$MAINPID
KillMode=mixed
Type=notify

Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

### HAProxy config

```bash
rm -f /etc/haproxy/*
```

#### HAProxy default config

```bash
cat <<EOF | tee /etc/haproxy/haproxy.cfg
# infra @ 10.1.1.11

#=====================================================================
# Global settings
# Document: https://www.haproxy.org/download/2.6/doc/configuration.txt
#=====================================================================
global
    daemon
    user        haproxy
    group       haproxy
    node        infra
    pidfile     /var/run/haproxy.pid
    #chroot     /var/lib/haproxy            # if chroot, change stats socket above
    stats socket /var/run/haproxy.sock mode 600 level admin
    # spread-checks 3                       # add randomness in check interval
    # quiet                                 # Do not display any message during startup
    maxconn     65535                       # maximum per-process number of concurrent connections

#---------------------------------------------------------------------
# default settings
#---------------------------------------------------------------------
defaults
    #log               global
    mode               tcp
    retries            3            # max retry connect to upstream
    timeout queue      3s           # maximum time to wait in the queue for a connection slot to be free
    timeout connect    3s           # maximum time to wait for a connection attempt to a server to succeed
    timeout client     24h          # client connection timeout
    timeout server     24h          # server connection timeout
    timeout check      3s           # health check timeout

#---------------------------------------------------------------------
# users and auth
#---------------------------------------------------------------------
userlist stats-auth
    group admin users admin
    user admin insecure-password kubegg
    group readonly users haproxy
    user haproxy insecure-password haproxy

#---------------------------------------------------------------------
# stats and exporter
#---------------------------------------------------------------------
# https://www.haproxy.com/blog/exploring-the-haproxy-stats-page/
listen stats                                # both frontend and a backend for statistics
    #option httplog                         # log http activity
    bind *:9101                             # default haproxy exporter port
    mode  http                              # server in http mode
    stats enable                            # enable stats page on http://127.0.0.1:9101
    stats hide-version
    stats uri /infra/
    stats refresh 30s                       # refresh stats page every 30 seconds
    stats show-node
    acl AUTH       http_auth(stats-auth)
    acl AUTH_ADMIN http_auth_group(stats-auth) admin
    stats http-request auth unless AUTH
    stats admin if AUTH_ADMIN
    http-request use-service prometheus-exporter if {path /metrics}

EOF
```

```bash
chmod 0644 /etc/haproxy/haproxy.cfg
```

#### HAProxy service config

```bash
# cat <<EOF | tee /etc/haproxy/{{ item.name}}.cfg
```

## HAProxy 启动

```bash
systemctl enable haproxy && \
/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c -q && \
systemctl restart haproxy
```

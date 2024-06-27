# Blackbox 安装

## Linux 主机准备

基于上一步安装完 Pushgateway 的主机，继续安装 Blackbox

## Blackbox 配置

### Blackbox exporter config

```bash
cat <<EOF | tee /etc/blackbox.yml
---
modules:
  icmp:
    prober: icmp
  icmp_ttl5:
    prober: icmp
    timeout: 5s
    icmp:
      ttl: 5
  http_2xx:
    prober: http
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
...

EOF
```

```bash
chown prometheus:prometheus /etc/blackbox.yml && \
chmod 0644 /etc/blackbox.yml
```

### Blackbox opts

```bash
cat <<EOF | tee /etc/default/blackbox_exporter
BLACKBOX_EXPORTER_OPTS="--config.file=/etc/blackbox.yml"

EOF
```

```bash
chown prometheus:prometheus /etc/default/blackbox_exporter && \
chmod 0755 /etc/default/blackbox_exporter
```

### Blackbox permission

```bash
# if os is debian
# setcap cap_net_raw+ep /usr/bin/blackbox_exporter
```

## Blackbox 启动

```bash
# port :9115
systemctl enable blackbox_exporter
systemctl restart blackbox_exporter
```

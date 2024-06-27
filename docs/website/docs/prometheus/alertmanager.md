# Alertmanager 安装

## Linux 主机准备

基于上一步安装完 Prometheus 的主机，继续安装 Alertmanager

## Alertmanager 配置

### Alertmanager systemd service

```bash
cat <<EOF | tee /usr/lib/systemd/system/alertmanager.service
[Unit]
Description=Prometheus Alertmanager.
Documentation=https://github.com/prometheus/alertmanager
After=network.target

[Service]
EnvironmentFile=-/etc/default/alertmanager
User=prometheus
ExecStart=/usr/bin/alertmanager \$ALERTMANAGER_OPTS
ExecReload=/bin/kill -HUP \$MAINPID
Restart=on-failure
RestartSec=5s
LimitNOFILE=16384


[Install]
WantedBy=multi-user.target

EOF
```

### Alertmanager config

```bash
cat <<EOF | tee /etc/alertmanager.yml
---
# TODO: CUSTOMIZE ACCORDING TO YOUR OWN ENVIRONMENT !!!

#--------------------------------------------------------------#
# Global
#--------------------------------------------------------------#
global:
  resolve_timeout: 5m


#--------------------------------------------------------------#
# Route
#--------------------------------------------------------------#
# https://prometheus.io/docs/alerting/latest/configuration/#route
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'


#--------------------------------------------------------------#
# Receivers
#--------------------------------------------------------------#
# https://prometheus.io/docs/alerting/latest/configuration/#receiver
receivers:
  - name: 'web.hook'
    # webhook_configs:
    #   - url: 'http://127.0.0.1:5001/'

...

EOF
```

```bash
chown prometheus:prometheus /etc/alertmanager.yml && \
chmod 0644 /etc/alertmanager.yml
```

### Alertmanager opts

```bash
cat <<EOF | tee /etc/default/alertmanager
ALERTMANAGER_OPTS="--config.file=/etc/alertmanager.yml --cluster.advertise-address=0.0.0.0:9093 --storage.path=/data/prometheus/alertmanager "

EOF
```

```bash
chown prometheus:prometheus /etc/default/alertmanager && \
chmod 0755 /etc/default/alertmanager
```

## Alertmanager 启动

```bash
systemctl enable alertmanager
systemctl restart alertmanager
```

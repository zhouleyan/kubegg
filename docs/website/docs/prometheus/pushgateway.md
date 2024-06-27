# Pushgateway 安装

## Linux 主机准备

基于上一步安装完 Alertmanager 的主机，继续安装 Pushgateway

## Pushgateway 配置

### Pushgateway systemd service

```bash
cat <<EOF | tee /usr/lib/systemd/system/pushgateway.service
[Unit]
Description=Prometheus push acceptor for ephemeral and batch jobs.
Documentation=https://github.com/prometheus/pushgateway
After=network.target

[Service]
EnvironmentFile=-/etc/default/pushgateway
User=prometheus
ExecStart=/usr/bin/pushgateway \$PUSHGATEWAY_OPTS
ExecReload=/bin/kill -HUP \$MAINPID
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target

EOF
```

### Pushgateway opts

```bash
cat <<EOF | tee /etc/default/pushgateway
PUSHGATEWAY_OPTS="--persistence.file=/data/prometheus/pushgateway.metrics --persistence.interval=1m"

EOF
```

```bash
chown prometheus:prometheus /etc/default/pushgateway && \
chmod 0755 /etc/default/pushgateway
```

## Pushgateway 启动

```bash
systemctl enable pushgateway
systemctl restart pushgateway
```

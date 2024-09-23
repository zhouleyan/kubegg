# Infra 监控配置

## 注册 HAProxy Nginx 服务

### 创建 nginx haproxy 配置路径

```bash
mkdir -p /etc/nginx/conf.d/haproxy
```

### 注册 haproxy upstream

```bash
cat <<EOF | tee /etc/nginx/conf.d/haproxy/upstream-infra.conf
upstream infra {
    server 10.1.1.11:9101 max_fails=0;
}
EOF
```

### 注册 haproxy location

```bash
cat <<EOF | tee /etc/nginx/conf.d/haproxy/location-infra.conf
location ^~/infra/ {
    proxy_pass http://infra;
    proxy_connect_timeout 1;
}
EOF
```

### 重载 nginx

```bash
nginx -T
systemctl daemon-reload
systemctl reload nginx
```

## 注册节点 VIP DNS

### 创建 dns 配置文件

```bash
cat <<EOF | tee /etc/hosts.d/infra
10.1.1.99 infra
EOF
```

### 重载 dnsmasq 生效

```bash
systemctl daemon-reload
systemctl restart dnsmasq
```

## node_exporter 配置 / 启动

### node_exporter service

```bash
cat <<EOF | tee /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=Prometheus exporter for machine metrics
Documentation=https://github.com/prometheus/node_exporter
After=network.target

[Service]
EnvironmentFile=-/etc/default/node_exporter
User=root
ExecStart=/usr/bin/node_exporter \$NODE_EXPORTER_OPTS
ExecReload=/bin/kill -HUP \$MAINPID
Restart=on-failure
RestartSec=5s

CPUQuota=30%
MemoryLimit=200M

[Install]
WantedBy=multi-user.target
EOF
```

### node_exporter config

```bash
# 该选项会启用 / 禁用一些指标收集器，请根据您的需要进行调整。
cat <<EOF | tee /etc/default/node_exporter
NODE_EXPORTER_OPTS="--web.listen-address=':9100' --web.telemetry-path='/metrics' --no-collector.softnet --no-collector.nvme --collector.tcpstat --collector.processes"
EOF
```

### node_exporter 启动

```bash
systemctl enable node_exporter
systemctl restart node_exporter
```

## keepalived_exporter 配置 / 启动

### keepalived_exporter service

```bash
cat <<EOF | tee /usr/lib/systemd/system/keepalived_exporter.service
[Unit]

Description=Prometheus exporter for Keepalived metrics
Documentation=https://github.com/gen2brain/keepalived_exporter
After=network.target


[Service]
EnvironmentFile=-/etc/default/keepalived_exporter
User=root
ExecStart=/usr/bin/keepalived_exporter \$KEEPALIVED_EXPORTER_OPTS
ExecReload=/bin/kill -HUP \$MAINPID
Restart=on-failure
RestartSec=5s


[Install]
WantedBy=multi-user.target
EOF
```

### keepalived_exporter config

```bash
cat <<EOF | tee /etc/default/keepalived_exporter
KEEPALIVED_EXPORTER_OPTS="--web.listen-address=':9650' --web.telemetry-path='/metrics'"
EOF
```

### keepalived_exporter 启动

```bash
systemctl enable keepalived_exporter
systemctl restart keepalived_exporter
```

## 注册 prometheus target

### node target

```yaml
cat <<EOF | tee /etc/prometheus/targets/node/10.1.1.11.yml
# 10.1.1.11
# node, haproxy, promtail
- labels: {ip: 10.1.1.11 , ins: infra , cls: infra}
  targets:
    - 10.1.1.11:9100
    - 10.1.1.11:9101
    - 10.1.1.11:9080

# keepalived
- labels: {ip: 10.1.1.11 , ins: infra , cls: infra, vip: 10.1.1.99}
  targets: [10.1.1.11:9650]
EOF
```

### ping target

```yaml
cat <<EOF | tee /etc/prometheus/targets/ping/10.1.1.11.yml
# 10.1.1.11
- labels: {ip: 10.1.1.11 , ins: infra , cls: infra}
  targets: [10.1.1.11]
EOF
```

```yaml
cat <<EOF | tee /etc/prometheus/targets/ping/10.1.1.99---10.1.1.11.yml
# 10.1.1.99@10.1.1.11
- labels: {ip: 10.1.1.11 , ins: infra , cls: infra, vip: 10.1.1.99, job: node-vip}
  targets: [10.1.1.99]
EOF
```

## Promtail 配置

### 创建 Promtail 配置路径

```bash
mkdir -p /etc/promtail
```

### Promtail service

```bash
rm -rf /etc/systemd/system/promtail.service

cat <<EOF | tee /usr/lib/systemd/system/promtail.service
[Unit]
Description=Promtail service
Documentation=https://grafana.com/docs/loki/latest/clients/promtail/
After=network.target

[Service]
User=root
ExecStart=/usr/bin/promtail -config.file=/etc/promtail/config.yml
ExecReload=/bin/kill -HUP \$MAINPID
TimeoutSec=60
Restart=on-failure
RestartSec=5s
LimitNOFILE=655360

[Install]
WantedBy=multi-user.target
EOF
```

### Promtail config

```yaml
cat <<EOF | tee /etc/promtail/config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 9097

positions:                     # location of position status file
  filename: /var/log/positions.yaml
  sync_period: 10s             # How often to update the positions file
  ignore_invalid_yaml: true    # Whether to ignore & later overwrite positions files that are corrupted

clients:
  - url:   http://10.1.1.11:3100/loki/api/v1/push
    external_labels:
      ip:  10.1.1.11
      cls: infra
      ins: infra

scrape_configs:

  ################################################################
  #                        Nodes Logs                            #
  ################################################################
  # collect /var/log/messages dmesg cron logs on all nodes
  - job_name: nodes
    static_configs:
      - targets:
          - localhost
        labels:
          src: syslog
          job: node
          __path__: /var/log/messages

      - targets:
          - localhost
        labels:
          src: dmesg
          job: node
          __path__: /var/log/dmesg

      - targets:
          - localhost
        labels:
          src: cron
          job: node
          __path__: /var/log/cron

  ################################################################
  #                       Infra Log                              #
  ################################################################
  # collect nginx logs on meta nodes
  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          src: nginx
          job: infra
          __path__: /var/log/nginx/*.log

      - targets:
          - localhost
        labels:
          src: nginx-error
          job: infra
          __path__: /var/log/nginx-error.log

    pipeline_stages:
      - match:
          selector: '{src="nginx"}'
          stages:
            - regex:
                # logline example: 127.0.0.1 - - [21/Apr/2020:13:59:45 +0000] "GET /?foo=bar HTTP/1.1" 200 612 "http://example.com/lekkebot.html" "curl/7.58.0"
                expression: '.*\[(?P<timestamp>.*)\].*'
            - timestamp:
                source: timestamp
                format: '02/Jan/2006:15:04:05 -0700'

      - match:
          selector: '{src="nginx-error"}'
          stages:
            - regex:
                # logline example: 127.0.0.1 - - [21/Apr/2020:13:59:45 +0000] "GET /?foo=bar HTTP/1.1" 200 612 "http://example.com/lekkebot.html" "curl/7.58.0"
                expression: '(?P<timestamp>\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2}).*'
            - timestamp:
                source: timestamp
                format: '2006/Jan/02 15:04:05'
EOF
```

## Promtail 安装 / 启动

```bash
systemctl enable promtail
systemctl restart promtail
```

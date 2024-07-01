# Prometheus 安装

## Linux 主机准备

基于上一步初始化的 Rocky8 主机，克隆一个主机用于 Prometheus 安装

| 系统            | 最低要求            | Prometheus 版本 |
| --------------- | ------------------- | --------------- |
| Rocky Linux 8.10 | CPU: 2core, RAM: 4G | prometheus2     |

## 确定系统信息

```bash
cat /etc/os-release

# grafana,loki,logcli,promtail,prometheus2,alertmanager,pushgateway
```

## 导入离线包

```bash
mkdir -p /tmp/kubegg/iso

# 从 artifact 主机上传 iso 文件
# /tmp/kubegg/el-8.10-x86_64-rpms.iso
scp el-8.10-x86_64-rpms.iso root@10.1.1.11:/tmp/kubegg
```

挂载 iso 文件

```bash
mount -t iso9660 -o loop /tmp/kubegg/el-8.10-x86_64-rpms.iso /tmp/kubegg/iso
```

新建本地源

```bash
# 备份原始源
mv /etc/yum.repos.d /etc/yum.repos.d.kubegg.bak

mkdir -p /etc/yum.repos.d

# 添加本地源
cat <<EOF | tee /etc/yum.repos.d/kubegg-local.repo
[base-local]
name=rpms-local

baseurl=file:///tmp/kubegg/iso

enabled=1

gpgcheck=0

EOF
```

```bash
yum clean all && yum makecache
```

## 安装 Prometheus

```bash
yum install grafana loki-2.9.8 logcli-2.9.8 promtail-2.9.8 prometheus2 alertmanager pushgateway node_exporter blackbox_exporter
```

## 卸载 iso 文件

```bash
umount /tmp/kubegg/iso
```

## 重置源

```bash
rm -rf /etc/yum.repos.d
mv /etc/yum.repos.d.kubegg.bak /etc/yum.repos.d
```

```bash
yum clean all && yum makecache
```

## 创建 Prometheus 用户

```bash
# 创建 Infra 用户组
groupadd -r infra

# 把用户 prometheus 添加进 infra 组
usermod -aG infra prometheus
```

## 创建 Prometheus 文件夹

```bash
mkdir -p /data/prometheus && \
chown prometheus:prometheus /data/prometheus && \
chmod 0700 /data/prometheus

mkdir -p /data/prometheus/data && \
chown prometheus:prometheus /data/prometheus/data && \
chmod 0700 /data/prometheus/data

# prometheus_sd_dir: /etc/prometheus/targets
mkdir -p /etc/prometheus && \
chown prometheus:prometheus /etc/prometheus && \
chmod 0750 /etc/prometheus

mkdir -p /etc/prometheus/bin && \
chown prometheus:prometheus /etc/prometheus/bin && \
chmod 0750 /etc/prometheus/bin

mkdir -p /etc/prometheus/rules && \
chown prometheus:prometheus /etc/prometheus/rules && \
chmod 0750 /etc/prometheus/rules

mkdir -p /etc/prometheus/targets && \
chown prometheus:prometheus /etc/prometheus/targets && \
chmod 0750 /etc/prometheus/targets

mkdir -p /etc/prometheus/targets/node && \
chown prometheus:prometheus /etc/prometheus/targets/node && \
chmod 0750 /etc/prometheus/targets/node

mkdir -p /etc/prometheus/targets/ping && \
chown prometheus:prometheus /etc/prometheus/targets/ping && \
chmod 0750 /etc/prometheus/targets/ping

mkdir -p /etc/prometheus/targets/etcd && \
chown prometheus:prometheus /etc/prometheus/targets/etcd && \
chmod 0750 /etc/prometheus/targets/etcd

mkdir -p /etc/prometheus/targets/infra && \
chown prometheus:prometheus /etc/prometheus/targets/infra && \
chmod 0750 /etc/prometheus/targets/infra

mkdir -p /etc/prometheus/targets/mysql && \
chown prometheus:prometheus /etc/prometheus/targets/mysql && \
chmod 0750 /etc/prometheus/targets/mysql
```

## Prometheus 配置

### Prometheus systemd service

```bash
cat <<EOF | tee /usr/lib/systemd/system/prometheus.service
[Unit]
Description=The Prometheus monitoring system and time series database.
Documentation=https://prometheus.io
After=network.target

[Service]
EnvironmentFile=-/etc/default/prometheus
User=prometheus
ExecStart=/usr/bin/prometheus \$PROMETHEUS_OPTS
ExecReload=/bin/kill -HUP \$MAINPID
Restart=always
RestartSec=5s
LimitNOFILE=16777216

[Install]
WantedBy=multi-user.target

EOF
```

### Prometheus opts

```bash
cat <<EOF | tee /etc/default/prometheus
PROMETHEUS_OPTS='--config.file=/etc/prometheus/prometheus.yml --web.page-title="Kubegg Metrics"--storage.tsdb.path=/data/prometheus/data --storage.tsdb.retention.time=15d'

EOF
```

```bash
chown prometheus:prometheus /etc/default/prometheus && \
chmod 0755 /etc/default/prometheus
```

### Prometheus config

```bash
cat <<EOF | tee /etc/prometheus/prometheus.yml
---
#--------------------------------------------------------------#
# Config FHS
#--------------------------------------------------------------#
# /etc/prometheus/
#  ^-----prometheus.yml    # prometheus main config file
#  ^-----alertmanager.yml  # alertmanger main config file
#  ^-----@bin              # util scripts: check,reload,status,new
#  ^-----@rules            # record & alerting rules definition
#
# {{prometheus_sd_dir}}
#            ^-----@node   # node static targets definition
#            ^-----@node   # ping targets definition
#            ^-----@pgsql  # pgsql static targets definition
#            ^-----@pgrds  # pgsql remote rds static targets
#            ^-----@redis  # redis static targets definition
#            ^-----@etcd   # etcd static targets definition
#            ^-----@minio  # minio static targets definition
#            ^-----@infra  # infra static targets definition
#            ^-----@mongo  # mongo static targets definition
#            ^-----@mysql  # mysql static targets definition
#--------------------------------------------------------------#


#--------------------------------------------------------------#
# Globals
#--------------------------------------------------------------#
global:
  scrape_interval: 10s
  evaluation_interval: 10s
  scrape_timeout: 8s


#--------------------------------------------------------------#
# Alerts
#--------------------------------------------------------------#
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['10.1.1.11:9093']
      scheme: http
      timeout: 10s
      api_version: v2


#--------------------------------------------------------------#
# Rules
#--------------------------------------------------------------#
rule_files:
  - rules/*.yml


#--------------------------------------------------------------#
# Targets Definition
#--------------------------------------------------------------#
# https://prometheus.io/docs/prometheus/latest/configuration/configuration/#file_sd_config
scrape_configs:


  #--------------------------------------------------------------#
  # job: prometheus
  # prometheus metrics
  #--------------------------------------------------------------#
  - job_name: prometheus
    metrics_path: /metrics
    honor_labels: true
    static_configs: [{targets: ['127.0.0.1:9090'] }]

  #--------------------------------------------------------------#
  # job: push
  # pushgateway metrics
  #--------------------------------------------------------------#
  - job_name: push
    metrics_path: /metrics
    honor_labels: true
    static_configs: [{targets: ['127.0.0.1:9091'] }]

  #--------------------------------------------------------------#
  # job: ping
  # blackbox exporter metrics
  #--------------------------------------------------------------#
  - job_name: ping
    metrics_path: /probe
    params: {module: [icmp]}
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/ping/*.yml]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115

  #--------------------------------------------------------------#
  # job: infra
  # targets: prometheus | grafana | altermanager | loki | nginx
  # labels: [ip, instance, type]
  # path: targets/infra/<ip>.yml
  #--------------------------------------------------------------#
  - job_name: infra
    metrics_path: /metrics
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/infra/*.yml]

  #--------------------------------------------------------------#
  # job: node
  # node_exporter, haproxy, docker, promtail
  # labels: [cls, ins, ip, instance]
  # path: targets/pgsql/<ip>.yml
  #--------------------------------------------------------------#
  - job_name: node
    metrics_path: /metrics
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/node/*.yml]


  #--------------------------------------------------------------#
  # job: mysql
  # labels: [cls, ins, ip, instance]
  # path: targets/mysql/<mysql_instance>.yml
  #--------------------------------------------------------------#
  - job_name: mysql
    metrics_path: /metrics
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/mysql/*.yml]


...
EOF
```

```bash
chown prometheus:prometheus /etc/prometheus/prometheus.yml && \
chmod 0644 /etc/prometheus/prometheus.yml
```

### Prometheus bin scripts

```bash
cat <<EOF | tee /etc/prometheus/bin/check
#!/bin/bash
promtool check config /etc/prometheus/prometheus.yml
EOF

cat <<EOF | tee /etc/prometheus/bin/new
#!/bin/bash

echo "destroy prometheus data and create a new one"
systemctl stop prometheus
rm -rf /data/prometheus/data/*
systemctl start prometheus

echo "prometheus recreated"
systemctl status prometheus
EOF

cat <<EOF | tee /etc/prometheus/bin/reload
#!/bin/bash
ps aux | grep prometheus | grep 'config.file' | awk '{print \$2}' | xargs -n1 kill -s SIGHUP
EOF

cat <<EOF | tee /etc/prometheus/bin/status
#!/bin/bash

systemctl status prometheus
ps aux | grep prometheus
EOF

chown -R prometheus:prometheus /etc/prometheus/bin && \
chmod -R 0755 /etc/prometheus/bin
```

### Prometheus rules

TODO: prometheus rules

### Prometheus 启动

```bash
systemctl enable prometheus
systemctl restart prometheus

systemctl daemon-reload
systemctl reload prometheus
```

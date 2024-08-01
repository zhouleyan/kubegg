# Prometheus 服务注册

## 导入节点配置

```bash
cat <<EOF | tee /etc/prometheus/targets/infra/infra-1.yml
---

#--------------------------------------------------------------#
# Nginx (Exporter @ 9113)
#--------------------------------------------------------------#
- labels: {ip: 10.1.1.11, ins: nginx-1, cls: nginx}
  targets: [10.1.1.11:9113]

#--------------------------------------------------------------#
# Prometheus (9090)
#--------------------------------------------------------------#
- labels: {ip: 10.1.1.11, ins: prometheus-1, cls: prometheus}
  targets: [10.1.1.11:9090]

#--------------------------------------------------------------#
# PushGateway (9091)
#--------------------------------------------------------------#
- labels: {ip: 10.1.1.11, ins: pushgateway-1, cls: pushgateway}
  targets: [10.1.1.11:9091]

#--------------------------------------------------------------#
# AlertManager (9093)
#--------------------------------------------------------------#
- labels: {ip: 10.1.1.11, ins: alertmanager-1, cls: alertmanager}
  targets: [10.1.1.11:9093]

#--------------------------------------------------------------#
# BlackboxExporter (9115)
#--------------------------------------------------------------#
- labels: {ip: 10.1.1.11, ins: blackbox-1, cls: blackbox}
  targets: [10.1.1.11:9115]

#--------------------------------------------------------------#
# Grafana (3000)
#--------------------------------------------------------------#
- labels: {ip: 10.1.1.11, ins: grafana-1, cls: grafana}
  targets: [10.1.1.11:3000]

#--------------------------------------------------------------#
# Loki (3100)
#--------------------------------------------------------------#
- labels: {ip: 10.1.1.11, ins: loki-1, cls: loki}
  targets: [10.1.1.11:3100]

...

EOF
```

```bash
chown prometheus:prometheus /etc/prometheus/targets/infra/infra-1.yml && \
chmod 0644 /etc/prometheus/targets/infra/infra-1.yml
```

## 重载 Prometheus

```bash
systemctl daemon-reload
systemctl reload prometheus
```

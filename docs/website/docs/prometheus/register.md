# Prometheus 服务注册

## 导入节点配置

```bash
cat <<EOF | tee /etc/prometheus/targets/infra/infra-1.yml
---

#--------------------------------------------------------------#
# Nginx (Exporter @ 9113)
#--------------------------------------------------------------#
- labels: {ip: 127.0.0.1, ins: nginx-1, cls: nginx}
  targets: [127.0.0.1:9113]

#--------------------------------------------------------------#
# Prometheus (9090)
#--------------------------------------------------------------#
- labels: {ip: 127.0.0.1, ins: prometheus-1, cls: prometheus}
  targets: [127.0.0.1:9090]

#--------------------------------------------------------------#
# PushGateway (9091)
#--------------------------------------------------------------#
- labels: {ip: 127.0.0.1, ins: pushgateway-1, cls: pushgateway}
  targets: [127.0.0.1:9091]

#--------------------------------------------------------------#
# AlertManager (9093)
#--------------------------------------------------------------#
- labels: {ip: 127.0.0.1, ins: alertmanager-1, cls: alertmanager}
  targets: [127.0.0.1:9093]

#--------------------------------------------------------------#
# BlackboxExporter (9115)
#--------------------------------------------------------------#
- labels: {ip: 127.0.0.1, ins: blackbox-1, cls: blackbox}
  targets: [127.0.0.1:9115]

#--------------------------------------------------------------#
# Grafana (3000)
#--------------------------------------------------------------#
- labels: {ip: 127.0.0.1, ins: grafana-1, cls: grafana}
  targets: [127.0.0.1:3000]

#--------------------------------------------------------------#
# Loki (3100)
#--------------------------------------------------------------#
- labels: {ip: 127.0.0.1, ins: loki-1, cls: loki}
  targets: [127.0.0.1:3100]

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

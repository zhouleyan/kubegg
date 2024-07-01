# Grafana 安装

## Linux 主机准备

基于上一步安装完 Blackbox 的主机，继续安装 Grafana

## 将 Grafana 用户添加进 infra 用户组

```bash
# 把用户 prometheus 添加进 infra 组
usermod -aG infra grafana
```

## 创建 Grafana 文件夹

```bash
mkdir -p /etc/grafana && \
chown grafana:grafana /etc/grafana && \
chmod 0775 /etc/grafana && \
mkdir -p /etc/dashboards && \
chown grafana:grafana /etc/dashboards && \
chmod 0775 /etc/dashboards && \
mkdir -p /etc/grafana/provisioning/dashboards && \
chown grafana:grafana /etc/grafana/provisioning/dashboards && \
chmod 0775 /etc/grafana/provisioning/dashboards && \
mkdir -p /etc/grafana/provisioning/datasources && \
chown grafana:grafana /etc/grafana/provisioning/datasources && \
chmod 0775 /etc/grafana/provisioning/datasources
```

## Grafana 配置

### datasources 配置

```bash
cat <<EOF | tee /etc/grafana/provisioning/datasources/kubegg.yml
---
apiVersion: 1

# remove provisioned data sources
deleteDatasources:
  - {name: Prometheus , orgId: 1}
  - {name: Meta       , orgId: 1}
  - {name: Loki       , orgId: 1}

# install following data sources
datasources:

  # Kubegg Monitor Database (Prometheus)
  - name: Prometheus
    uid: ds-prometheus
    type: prometheus
    url: http://127.0.0.1:9090
    access: proxy
    isDefault: true
    editable: true
    version: 1
    jsonData:
      timeInterval: "2s"
      queryTimeout: "60s"
      tlsAuth: false
      tlsAuthWithCACert: false
    secureJsonData: {}

  # Kubegg Logging Database (Loki)
  - name: Loki
    type: loki
    uid: ds-loki
    url: http://127.0.0.1:3100
    access: proxy
    editable: true
...
EOF

chown grafana:grafana /etc/grafana/provisioning/datasources/kubegg.yml
```

### grafana.ini 配置

```bash title="/etc/grafana/grafana.ini"
sed -i 's/^admin_user = .*/admin_user = admin/g' /etc/grafana/grafana.ini && \
sed -i 's/^admin_password = .*/admin_password = kubegg/g' /etc/grafana/grafana.ini
```

### dashboards 配置

```bash
cat <<EOF | tee /etc/grafana/provisioning/dashboards/kubegg.yml
---
# load dashboards from /etc/dashboards
apiVersion: 1
providers:

  - name: Kubegg
    orgId: 1
    type: file
    updateIntervalSeconds: 8
    allowUiUpdates: true
    options:
      path: /etc/dashboards/
      foldersFromFilesStructure: true
...
EOF

chown grafana:grafana /etc/grafana/provisioning/dashboards/kubegg.yml
```

### grafana logo 自定义

```bash
# 替换 /usr/share/grafana/public/img/grafana_icon.svg
```

## Grafana 插件安装

```bash
grafana-cli plugins install volkovlabs-echarts-panel && \
grafana-cli plugins install volkovlabs-image-panel && \
grafana-cli plugins install volkovlabs-form-panel && \
grafana-cli plugins install volkovlabs-variable-panel && \
grafana-cli plugins install volkovlabs-grapi-datasource && \
grafana-cli plugins install marcusolsson-static-datasource && \
grafana-cli plugins install marcusolsson-json-datasource && \
grafana-cli plugins install marcusolsson-dynamictext-panel && \
grafana-cli plugins install marcusolsson-treemap-panel && \
grafana-cli plugins install marcusolsson-calendar-panel && \
grafana-cli plugins install marcusolsson-hourly-heatmap-panel && \
grafana-cli plugins install knightss27-weathermap-panel
```

## Grafana 启动

```bash
systemctl enable grafana-server
systemctl restart grafana-server
```

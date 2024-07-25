# Infra 初始化

## Linux 主机准备

基于上一步初始化的 Rocky8 主机，克隆一个主机用于 Infra 软件初始化

## 确定系统信息

```bash
cat /etc/os-release
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
mv /etc/yum.repos.d /etc/yum.repos.d.kubegg.bak && \
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

## 安装 infra 软件包

```bash
yum install grafana loki logcli promtail prometheus2 alertmanager pushgateway nginx node_exporter blackbox_exporter nginx_exporter dnsmasq haproxy keepalived keepalived_exporter iotop htop netcat socat audit lrzsz etcd readline tuned
```

## 卸载 iso 文件

```bash
umount /tmp/kubegg/iso && \
rm -rf /tmp/kubegg
```

## 重置源

```bash
rm -rf /etc/yum.repos.d && \
mv /etc/yum.repos.d.kubegg.bak /etc/yum.repos.d
```

```bash
yum clean all && yum makecache
```

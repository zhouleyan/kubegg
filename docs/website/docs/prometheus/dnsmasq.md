# Dnsmasq 安装

## 创建 DNS 记录文件路径

```bash
mkdir -p /etc/hosts.d && \
chown dnsmasq:dnsmasq /etc/hosts.d && \
chmod 0755 /etc/hosts.d
```

## DNS 配置

### dnsmasq service

```bash
cat <<EOF | tee /usr/lib/systemd/system/dnsmasq.service
[Unit]
Description=DNS caching server.
Wants=network-online.target
After=network-online.target

[Service]
Type=forking
PIDFile=/run/dnsmasq.pid
ExecStart=/usr/sbin/dnsmasq
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target

EOF
```

### dnsmasq config

```bash
cat <<EOF | tee /etc/dnsmasq.d/kubegg.conf
hostsdir=/etc/hosts.d
no-hosts
listen-address=127.0.0.1,10.1.1.11
port=53

EOF
```

### dns record

```bash
# 备份 resolv.conf
mv /etc/resolv.conf /etc/resolv.conf.bk

cat <<EOF | tee /etc/hosts.d/default
10.1.1.11 h.kubegg a.kubegg p.kubegg g.kubegg

EOF
```

```bash
chmod 0644 /etc/hosts.d/default
```

## 启动 dnsmasq

```bash
# port :53
systemctl enable dnsmasq
systemctl restart dnsmasq
```

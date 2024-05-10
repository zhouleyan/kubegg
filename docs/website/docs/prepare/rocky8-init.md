# Rocky8 初始化

## Linux 主机准备

选择 Rocky Linux 8.9 操作系统

| 系统            | 最低要求            |
| --------------- | ------------------- |
| Rocky Linux 8.9 | CPU: 2core, RAM: 4G |

### 修改 hostname

```bash
# 修改 hostname
hostnamectl set-hostname rocky8
```

### 网络配置

```bash
# 生成 uuid
uuidgen

# 生成 e11ffc43-5f46-487a-94f5-73c603924e88，替换下面网络 UUID

vi /etc/sysconfig/network-scripts/ifcfg-ens160

# 修改静态 IP 等配置

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=eui64
NAME=ens160
UUID=3fede721-5bc0-4043-bfe6-9597424ddc79
DEVICE=ens160
ONBOOT=yes
IPADDR=10.1.1.10
PREFIX=24
GATEWAY=10.1.1.2
DNS1=10.1.1.2
DNS2=8.8.8.8
DNS3=8.8.4.4

# 重启网卡生效
systemctl restart NetworkManager

# or
nmcli networking off
nmcli networking on

```

### Yum 更换国内源

```bash
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
    -i.bak \
    /etc/yum.repos.d/Rocky-*.repo

dnf makecache

```

### 更新包

```bash
yum update
yum autoremove
```

### 删除默认安装软件

```bash
yum remove firewalld python-firewall firewalld-filesystem
```

### 安装常用软件

```bash
yum update
yum clean all && yum makecache

yum install vim sudo curl wget bind-utils lz4 bash-completion net-tools tcpdump tree telnet openssl tar nss nss-sysinit nss-tools chrony mlocate sysstat iputils psmisc rsync libseccomp ebtables iptables ethtool nfs-utils glusterfs-client jq conntrack conntrack-tools socat ipset ipvsadm
```

### 禁用系统 swap

禁用系统 swap

```bash
source /etc/profile
swapoff -a && sysctl -w vm.swappiness=0

sed -i '/^[^#]*swap.*/s/^/\# /g' /etc/fstab
```

### 禁用 SELinux

查看 `/etc/selinux/config` 文件

```bash
cat /etc/selinux/config
```

如果存在 `/etc/selinux/config` 则关闭 SELinux

```bash
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

setenforce 0
getenforce
```

### journal 日志优化

优化 journal 设置，避免日志重复搜集，浪费系统资源

准备 journal 日志相关目录

```bash
mkdir -p /etc/systemd/journald.conf.d && \
mkdir -p /var/log/journal
```

优化设置 journal 日志

```bash title="/etc/systemd/journald.conf.d/95-rocky8-journald.conf"
cat>/etc/systemd/journald.conf.d/95-rocky8-journald.conf<<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent

# 最大占用空间 2G
SystemMaxUse=2G

# 单日志文件最大 200M
SystemMaxFileSize=200M

# 日志保存时间 2 周
MaxRetentionSec=2week

# 禁止转发
ForwardToSyslog=no
ForwardToWall=no
EOF
```

重启 journald 服务

```bash
systemctl restart systemd-journald
```

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
# systemctl restart NetworkManager

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

yum install vim sudo curl wget bind-utils lz4 bash-completion net-tools tcpdump tree telnet openssl tar nss nss-sysinit nss-tools chrony mlocate sysstat iputils psmisc rsync libseccomp ebtables iptables ethtool nfs-utils glusterfs-client jq conntrack conntrack-tools socat ipset ipvsadm yum-utils
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
cat <<EOF | sudo tee /etc/systemd/journald.conf.d/95-rocky8-journald.conf
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

### 加载内核模块

```bash
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack
modprobe overlay
modprobe ip_tables
modprobe ip_set
modprobe xt_set
modprobe ipt_set
modprobe ipt_rpfilter
modprobe ipt_REJECT
modprobe ipip

modprobe -a br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack overlay ip_tables ip_set xt_set ipt_set ipt_rpfilter ipt_REJECT ipip
```

查看已加载内核模块

```bash
lsmod | grep ip_vs
...
```

增加内核模块开机加载配置

```bash title="/etc/modules-load.d/10-rocky8-modules.conf"
cat <<EOF | sudo tee /etc/modules-load.d/10-rocky8-modules.conf
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
overlay
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
```

启用 systemd 自动加在模块服务

```bash
systemctl enable systemd-modules-load
```

## 系统调优

### 修改内核参数

添加内核参数配置文件

```bash title="/etc/sysctl.d/rocky8.conf"
cat <<EOF | sudo tee /etc/sysctl.d/rocky8.conf
net.ipv4.ip_forward                                      = 1
net.ipv4.ip_local_reserved_ports                         = 30000-32767
net.ipv4.tcp_retries2                                    = 15
net.ipv4.tcp_tw_reuse                                    = 0
net.ipv4.tcp_keepalive_time                              = 600
net.ipv4.tcp_keepalive_intvl                             = 30
net.ipv4.tcp_keepalive_probes                            = 10
net.ipv4.tcp_max_syn_backlog                             = 1048576
net.ipv4.tcp_max_tw_buckets                              = 1048576
net.ipv4.tcp_max_orphans                                 = 65535
net.ipv4.udp_rmem_min                                    = 131072
net.ipv4.udp_wmem_min                                    = 131072
net.ipv4.neigh.default.gc_thresh1                        = 512
net.ipv4.neigh.default.gc_thresh2                        = 2048
net.ipv4.neigh.default.gc_thresh3                        = 4096
net.ipv4.conf.all.rp_filter                              = 1
net.ipv4.conf.default.rp_filter                          = 1
net.ipv4.conf.all.arp_accept                             = 1
net.ipv4.conf.default.arp_accept                         = 1
net.ipv4.conf.all.arp_ignore                             = 1
net.ipv4.conf.default.arp_ignore                         = 1
net.bridge.bridge-nf-call-arptables                      = 1
net.bridge.bridge-nf-call-ip6tables                      = 1
net.bridge.bridge-nf-call-iptables                       = 1
net.core.netdev_max_backlog                              = 65535
net.core.rmem_max                                        = 33554432
net.core.wmem_max                                        = 33554432
net.core.somaxconn                                       = 32768
net.netfilter.nf_conntrack_max                           = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established       = 3600
vm.max_map_count                                         = 655360
vm.swappiness                                            = 0
vm.overcommit_memory                                     = 1
fs.file-max                                              = 6553600
fs.inotify.max_user_instances                            = 524288
fs.inotify.max_user_watches                              = 10240001
fs.pipe-max-size                                         = 4194304
fs.aio-max-nr                                            = 262144
kernel.pid_max                                           = 65535
kernel.watchdog_thresh                                   = 5
kernel.hung_task_timeout_secs                            = 5
EOF
```

生效内核参数修改

```bash
sysctl --system

# or
# sysctl -p 不支持在 sysctl.d 内配置文件。
```

验证内核参数已被修改，例如：`net.ipv4.ip_forward`、`net.bridge.bridge-nf-call-iptables`

```bash
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
```

### ulimit 调优

修改 limits 配置

```bash title="/etc/security/limits.d/30-rocky8.conf"
cat <<EOF | sudo tee /etc/security/limits.d/10-rocky8.conf
root soft nofile    1048576
root hard nofile    1048576
root soft nproc     65536
root hard nproc     65536
root soft memlock   unlimited
root hard memlock   unlimited
root soft core      unlimited
root hard core      unlimited
*    soft nofile    1048576
*    hard nofile    1048576
*    soft nproc     65536
*    hard nproc     65536
*    soft memlock   unlimited
*    hard memlock   unlimited
*    soft core      unlimited
*    hard core      unlimited
EOF
```

验证 limits 配置生效

```bash
# 分别在 root 以级其他用户下验证。
ulimit -a
```

清理内存缓存

```bash
sync
echo 3 > /proc/sys/vm/drop_caches
```

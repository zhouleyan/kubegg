# OS 初始化

## Linux 主机准备

选择 Debian 操作系统

| 系统        | 最低要求            |
| ----------- | ------------------- |
| Debian 12.3 | CPU: 2core, RAM: 4G |

安装系统时，选择最小化安装 + ssh-server。

使用 root 账户登录。

### vi 设置

```bash title="/etc/vim/vimrc.tiny"
vi /etc/vim/vimrc.tiny
:set nocp

# Add below...

set nocompatible
set number
set t_Co=256
set backspace=2
```

### sshd 配置

> 允许 root 用户通过 ssh 登录（不建议）。

```bash title="/etc/ssh/sshd_config"
sed -i 's/PermitRootLogin no/PermitRootLogin yes/g' /etc/ssh/sshd_config

systemctl restart sshd
```

### 修改主机名

修改 hostname。

```bash
# 修改 hostname
hostnamectl set-hostname master1
```

修改 domain。

```bash title="/etc/resolv.conf"
search master1
nameserver 172.16.239.2
nameserver 8.8.8.8
nameserver 8.8.4.4
```

修改 hosts。

```bash title="/etc/hosts"
127.0.0.1       localhost
172.16.239.111  master1.master1 master1

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### 网络配置

使用 root 登录后，修改 `/etc/network/interfaces` 文件。

```bash title="/etc/network/interfaces"
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens33
iface ens33 inet static
        address 172.16.239.111/24
        gateway 172.16.239.2
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 172.16.239.2 8.8.8.8 8.8.4.4
        dns-search master1
```

重启网络。

```bash
systemctl restart networking
```

查看 `uuid` 是否唯一。

```bash
cat /sys/class/dmi/id/product_uuid
5cc44d56-d5d4-c3ab-ee58-96825f0f1d93
```

修改 mac 地址（可选）。

使用 [mac-address-generator](https://miniwebtool.com/mac-address-generator/) 生产 mac 地址。

`00:0c:29:48:08:31`

```bash
ip link set ens33 down
ip link set ens33 address 00:0c:29:48:08:31
ip link set ens33 up
```

### 确认系统使用 Cgroup v2

```bash
stat -fc %T /sys/fs/cgroup/
cgroup2fs
```

### apt 更换国内源

打开 `/etc/apt/sources.list` 文件，替换其中所有内容为国内源地址。

```bash
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware

deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contrib non-free non-free-firmware
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contrib non-free non-free-firmware
```

### 安装常用软件

```bash
apt update
apt upgrade -y
apt -y install vim sudo curl wget bash-completion net-tools openssl tar apt-transport-https ca-certificates gpg chrony
apt autoremove
```

### 安装 Kubernetes 依赖软件

```bash
apt update

# 删除默认安装
apt remove ufw lxd lxcfs lxc-common

# 安装软件包
apt install conntrack          # network connection cleanup 用到
apt install ipset              # ipvs 模式需要
apt install ipvsadm            # ipvs 模式需要
apt install jq                 # 轻量 JSON 处理程序，安装 Docker/containerd 查询镜像需要
apt install libseccomp2        # 安装 containerd 需要
apt install nfs-common         # 挂载 nfs 共享文件需要（创建基于 nfs 的 PV 需要）
apt install ceph-common        # 挂载 ceph 共享文件需要（创建基于 ceph 的 PV 需要）
apt install glusterfs-client   # 挂载 glusterfs 共享文件需要（创建基于 glusterfs 的 PV 需要）
apt install psmisc             # 安装 psmisc 才能使用命令 killall，keepalive 的监测脚本需要
apt install rsync              # 文件同步工具，分发证书等配置文件需要
apt install socat              # 用于 port forwarding
apt install ebtables
apt install iptables
apt install ethtool

# apt install conntrack ipset ipvsadm jq libseccomp2 nfs-common ceph-common glusterfs-client psmisc rsync socat ebtables iptables ethtool
```

### 禁用系统 swap

禁用系统 swap。

```bash
source /etc/profile
swapoff -a && sysctl -w vm.swappiness=0

sed -i /^[^#]*swap.*/s/^/\# /g /etc/fstab
```

### 禁用 SELinux

查看 `/etc/selinux/config` 文件

```bash
cat /etc/selinux/config
```

如果存在 `/etc/selinux/config`，关闭 SELinux。

```bash
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

setenforce 0
getenforce
```

### journal 日志优化

优化 journal 设置，避免日志重复搜集，浪费系统资源。

准备 journal 日志相关目录。

```bash
mkdir -p /etc/systemd/journald.conf.d
mkdir -p /var/log/journal
```

优化设置 journal 日志。

```bash title="/etc/systemd/journald.conf.d/95-k8s-journald.conf"
cat>/etc/systemd/journald.conf.d/95-k8s-journald.conf<<EOF
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
```

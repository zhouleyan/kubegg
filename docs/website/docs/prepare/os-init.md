# OS 初始化

## Linux 主机准备

选择 Debian 操作系统

| 系统        | 最低要求            |
| ----------- | ------------------- |
| Debian 12.3 | CPU: 2core, RAM: 4G |

安装系统时，选择最小化安装 + ssh-server

使用 root 账户登录

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
sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/g' /etc/ssh/sshd_config

systemctl restart sshd
```

### ll 命令配置

```bash
sudo echo "alias ll='ls -l'" > /etc/profile.d/ll.sh
source /etc/profile
```

### 修改主机名

修改 hostname

```bash
# 修改 hostname
hostnamectl set-hostname master1
```

修改 domain

```bash title="/etc/resolv.conf"
search master1
nameserver 172.16.239.2
nameserver 8.8.8.8
nameserver 8.8.4.4
```

修改 hosts

```bash title="/etc/hosts"
127.0.0.1       localhost
172.16.239.111  master1.master1 master1

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### 网络配置

使用 root 登录后，修改 `/etc/network/interfaces` 文件

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

重启网络

```bash
systemctl restart networking
```

修改 mac 地址（可选）

查看网卡 mac 地址

```bash
cat /sys/class/net/<network_interface>/address
```

使用 [mac-address-generator](https://miniwebtool.com/mac-address-generator/) 生产 mac 地址

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

打开 `/etc/apt/sources.list` 文件，替换其中所有内容为国内源地址

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
apt -y install vim sudo curl wget bash-completion net-tools telnet openssl tar apt-transport-https ca-certificates gpg chrony locate sysstat
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
mkdir -p /etc/systemd/journald.conf.d
mkdir -p /var/log/journal
```

优化设置 journal 日志

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
modprobe ip_tables
modprobe ip_set
modprobe xt_set
modprobe ipt_set
modprobe ipt_rpfilter
modprobe ipt_REJECT
modprobe ipip
```

查看已加载内核模块

```bash
lsmod | grep ip_vs
...
```

增加内核模块开机加载配置

```bash title="/etc/modules-load.d/10-k8s-modules.conf"
cat>/etc/modules-load.d/10-k8s-modules.conf<<EOF
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

### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
systemctl stop ufw
systemctl disable ufw
```

## 系统调优

### 修改内核参数

添加内核参数配置文件

```bash title="/etc/sysctl.d/k8s.conf"
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
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

```bash title="/etc/security/limits.d/30-k8s.conf"
cat <<EOF | sudo tee /etc/security/limits.d/10-k8s.conf
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

## 预分配系统用户与文件夹

### 创建系统用户

```bash
useradd -M -c 'Kubernetes user' -s /sbin/nologin -r kube || :
useradd -M -c 'Etcd user' -s /sbin/nologin -r etcd || :
```

### 创建 Kubernetes 所需文件夹

* `/usr/local/bin`
* `/etc/kubernetes`
* `/etc/kubernetes/pki`
* `/etc/kubernetes/manifests`
* `/usr/local/bin/kube-scripts`
* `/usr/libexec/kubernetes/kubelet-plugins/volume/exec`
* `/etc/cni/net.d`
* `/opt/cni/bin`
* `/var/lib/calico`
* `/var/lib/etcd`

```bash
mkdir -p /usr/local/bin && chown kube -R /usr/local/bin
mkdir -p /etc/kubernetes && chown kube -R /etc/kubernetes
mkdir -p /etc/kubernetes/pki && chown kube -R /etc/kubernetes/pki
mkdir -p /etc/kubernetes/manifests && chown kube -R /etc/kubernetes/manifests
mkdir -p /usr/local/bin/kube-scripts && chown kube -R /usr/local/bin/kube-scripts
mkdir -p /usr/libexec/kubernetes/kubelet-plugins/volume/exec && chown kube -R /usr/libexec/kubernetes
mkdir -p /etc/cni/net.d && chown kube -R /etc/cni
mkdir -p /opt/cni/bin && chown kube -R /opt/cni
mkdir -p /var/lib/calico && chown kube -R /var/lib/calico
mkdir -p /var/lib/etcd && chown etcd -R /var/lib/etcd
```

## 安装 containerd

安装 `containerd` 二进制文件至 `/usr/bin`，版本：`1.7.9`

安装 `crictl` 二进制文件至 `/usr/bin` 版本：`v1.28.0`

创建临时文件夹 `/tmp/kubegg`

```bash
mkdir /tmp/kubegg
```

### 安装 containerd 二进制文件

下载地址：[https://github.com/containerd/containerd/releases/download/v1.7.9/containerd-1.7.9-linux-amd64.tar.gz](https://github.com/containerd/containerd/releases/download/v1.7.9/containerd-1.7.9-linux-amd64.tar.gz)

下载 `containerd`

```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.9/containerd-1.7.9-linux-amd64.tar.gz -O /tmp/kubegg/containerd-1.7.9-linux-amd64.tar.gz
```

安装 `containerd` 二进制文件

```bash
mkdir -p /usr/bin && tar -zxf /tmp/kubegg/containerd-1.7.9-linux-amd64.tar.gz -C /tmp/kubegg
mv /tmp/kubegg/bin/* /usr/bin
rm -rf /tmp/kubegg/bin
```

### 安装 crictl 二进制文件

下载地址：[https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz](https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz)

下载 `crictl`

```bash
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz -O /tmp/kubegg/crictl-v1.28.0-linux-amd64.tar.gz
```

安装 `crictl` 二进制文件

```bash
mkdir -p /usr/bin && tar -zxf /tmp/kubegg/crictl-v1.28.0-linux-amd64.tar.gz -C /usr/bin
```

### 安装 runc 二进制文件

下载地址：[https://github.com/opencontainers/runc/releases/download/v1.1.10/runc.amd64](https://github.com/opencontainers/runc/releases/download/v1.1.10/runc.amd64)

下载 `runc`

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.10/runc.amd64 -O /tmp/kubegg/runc.amd64
```

安装 `runc` 二进制文件

```bash
install -m 755 /tmp/kubegg/runc.amd64 /usr/local/sbin/runc
```

### 安装 cni 二进制文件

下载地址：[https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz](https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz)

下载 `cni`

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz -O /tmp/kubegg/cni-plugins-linux-amd64-v1.3.0.tgz
```

安装 `cni` 二进制文件

```bash
tar -zxf /tmp/kubegg/cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin
```

### 创建 containerd systemd unit 文件

```bash title="/etc/systemd/system/containerd.service"
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

### 创建 containerd 配置文件

创建 `containerd` 配置目录

```bash
mkdir -p /etc/containerd
```

生成 `containerd` 配置文件

```bash
containerd config default > /etc/containerd/config.toml
```

```bash title="/etc/containerd/config.toml"
cat <<EOF | sudo tee /etc/containerd/config.toml
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
temp = ""
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_ca = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    cdi_spec_dirs = ["/etc/cdi", "/var/run/cdi"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    drain_exec_sync_io_timeout = "0s"
    enable_cdi = false
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_image_defined_volumes = false
    image_pull_progress_timeout = "1m0s"
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "registry.kubegg.local/pause:3.8"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
      ip_pref = ""
      max_conf_num = 1
      setup_serially = false

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      ignore_blockio_not_enabled_errors = false
      ignore_rdt_not_enabled_errors = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        privileged_without_host_devices_all_devices_allowed = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""
        sandbox_mode = ""
        snapshotter = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          cni_conf_dir = ""
          cni_max_conf_num = 0
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          privileged_without_host_devices_all_devices_allowed = false
          runtime_engine = ""
          runtime_path = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          sandbox_mode = "podsandbox"
          snapshotter = ""

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        privileged_without_host_devices_all_devices_allowed = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""
        sandbox_mode = ""
        snapshotter = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://docker.nju.edu.cn/", "https://kuamavit.mirror.aliyuncs.com"]

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
          endpoint = ["https://gcr.nju.edu.cn"]

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://gcr.nju.edu.cn/google-containers/"]

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
          endpoint = ["https://quay.nju.edu.cn"]

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."ghcr.io"]
          endpoint = ["https://ghcr.nju.edu.cn"]

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."nvcr.io"]
          endpoint = ["https://ngc.nju.edu.cn"]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.internal.v1.tracing"]
    sampling_ratio = 1.0
    service_name = "containerd"

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.nri.v1.nri"]
    disable = true
    disable_connections = false
    plugin_config_path = "/etc/nri/conf.d"
    plugin_path = "/opt/nri/plugins"
    plugin_registration_timeout = "5s"
    plugin_request_timeout = "2s"
    socket_path = "/var/run/nri/nri.sock"

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
    sched_core = false

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.service.v1.tasks-service"]
    blockio_config_file = ""
    rdt_config_file = ""

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.blockfile"]
    fs_type = ""
    mount_options = []
    root_path = ""
    scratch_file = ""

  [plugins."io.containerd.snapshotter.v1.btrfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    discard_blocks = false
    fs_options = ""
    fs_type = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    mount_options = []
    root_path = ""
    sync_remove = false
    upperdir_label = false

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

  [plugins."io.containerd.tracing.processor.v1.otlp"]
    endpoint = ""
    insecure = false
    protocol = ""

  [plugins."io.containerd.transfer.v1.local"]
    config_path = ""
    max_concurrent_downloads = 3
    max_concurrent_uploaded_layers = 3

    [[plugins."io.containerd.transfer.v1.local".unpack_config]]
      differ = ""
      platform = "linux/amd64"
      snapshotter = "overlayfs"

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.bolt.open" = "0s"
  "io.containerd.timeout.metrics.shimstats" = "2s"
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0
EOF
```

### 生成 crictl 配置文件

```bash title="/etc/crictl.yaml"
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 5
debug: false
pull-image-on-create: false
EOF
```

### 启动 containerd

启动 `containerd` 服务，并设置开机自动启动

```bash
systemctl daemon-reload && systemctl enable containerd && systemctl start containerd
```

## 安装其他二进制文件

### 安装 cfssl

下载地址：[https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64](https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64)

下载地址：[https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64](https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64)

下载 `cfssl`、`cfssljson`

```bash
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64 -O /usr/local/bin/cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64 -O /usr/local/bin/cfssljson

chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
```

### 安装 kubectl

下载地址：[https://dl.k8s.io/release/v1.28.4/bin/linux/amd64/kubectl](https://dl.k8s.io/release/v1.28.4/bin/linux/amd64/kubectl)

下载 `kubectl`

```bash
wget https://dl.k8s.io/release/v1.28.4/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl && chmod +x /usr/local/bin/kubectl
```

开启 `kubectl` 自动补全

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc

# alias
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

### 安装 helm

下载地址：[https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz](https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz)

下载 `helm`

```bash
wget https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz -O /tmp/kubegg/helm-v3.13.2-linux-amd64.tar.gz
```

安装 `helm` 二进制文件

```bash
tar -zxf /tmp/kubegg/helm-v3.13.2-linux-amd64.tar.gz -C /tmp/kubegg

mv /tmp/kubegg/linux-amd64/helm /usr/local/bin

rm -rf /tmp/kubegg/linux-amd64

chmod +x /usr/local/bin/helm
```

### 安装 calicoctl

下载地址：[https://github.com/projectcalico/calico/releases/download/v3.26.4/calicoctl-linux-amd64](https://github.com/projectcalico/calico/releases/download/v3.26.4/calicoctl-linux-amd64)

下载 `calicoctl`

```bash
wget https://github.com/projectcalico/calico/releases/download/v3.26.4/calicoctl-linux-amd64 -O /usr/local/bin/calicoctl
```

安装 `calicoctl` 二进制文件

```bash
chmod +x /usr/local/bin/calicoctl
```

截止此，基础 OS 初始化完毕，后续可基于该 OS 镜像克隆。

# 节点初始化

## Linux 主机准备

基于上一步初始化的 Rocky8 主机，克隆一个主机用于 MySQL 安装

| 系统             | 最低要求            | MySQL 版本 |
| ---------------- | ------------------- | ---------- |
| Rocky Linux 8.10 | CPU: 2core, RAM: 4G | 16.4       |

## 确定系统信息

```bash
cat /etc/os-release
```

## 安装顺序

- [node_id] get node identity
- [pg_id] get pgsql identity
- [node] node init
- [haproxy] install haproxy
- [node_monitor] node monitor
- [pgsql] pgsql init

## hostname 设置

```bash
hostnamectl set-hostname pg16-master01
```

## DNS 设置

修改 `/etc/hosts`

```bash
sudo sed -i '1s/^/10.1.1.99 h.kubegg a.kubegg p.kubegg g.kubegg h.kubegg.local a.kubegg.local p.kubegg.local g.kubegg.local # kubegg dns\n/' /etc/hosts
```

修改 `/etc/resolv.conf`

```bash
cat <<EOF | sudo tee /etc/resolv.conf
options single-request-reopen timeout:1
nameserver 10.1.1.11

nameserver 10.1.1.2
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF
```

## 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld

# 关闭 selinux
setenforce 0
sed -ie "s/^SELINUX=.*/SELINUX=disabled/g" /etc/selinux/config
rm -rf /etc/selinux/confige
```

## 上传 CA 证书

从 `infra` 节点上传 CA 证书

```bash
mkdir -p /etc/pki

# infra exec
scp /etc/pki/ca/ca.pem root@10.1.1.31:/etc/pki/ca.pem

rm -rf /etc/pki/ca-trust/source/anchors/ca.pem && \
ln -s /etc/pki/ca.pem /etc/pki/ca-trust/source/anchors/ca.pem

/bin/update-ca-trust
```

## 安装默认包

```bash
sudo dnf install lz4 unzip bzip2 zlib yum pv jq git ncdu make patch bash lsof wget uuid tuned nvme-cli numactl grubby sysstat iotop htop rsync tcpdump chrony python3 netcat socat ftp lrzsz net-tools ipvsadm bind-utils telnet audit ca-certificates openssl readline vim-minimal node_exporter etcd haproxy python3-pip
```

## 系统功能优化

### 功能优化

```bash
# 关闭 NUMA
# disable numa
grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="numa=off transparent_hugepage=never"
```

```bash
# disable swap
source /etc/profile
swapoff -a && sysctl -w vm.swappiness=0
sed -i '/^[^#]*swap.*/s/^/\# /g' /etc/fstab
```

```bash
# 使用静态 DNS 服务器
# static_network
cat <<EOF | sudo tee ~/static_network.sh
for interface in \$(ls /etc/sysconfig/network-scripts/ifcfg-*); do
    if (! grep -q 'PEERDNS' \$interface); then
        echo 'PEERDNS=no' >> \$interface
    else
        sed -i s/PEERDNS=.*/PEERDNS=no/ \$interface;
    fi
done
EOF

chmod +x ~/static_network.sh

sh ~/static_network.sh
```

```bash
# 启用磁盘预读
# setup disk prefetch
cat <<EOL | sudo tee ~/disk_prefetch.sh
#!/usr/bin/env bash

# setup disk prefetch
if (! grep -q 'disk prefetch' /etc/rc.local); then
    cat >>/etc/rc.local <<-EOF
# disk prefetch
blockdev --setra 16384 \$(echo \$(blkid | awk -F':' '\$1!~"block"{print \$1}'))
EOF
    chmod +x /etc/rc.d/rc.local
fi
exit 0
EOL

chmod +x ~/disk_prefetch.sh
sh ~/disk_prefetch.sh
```

### 内核模块

```bash
# 加载内核模块
# softdog, br_netfilter, ip_vs, ip_vs_rr, ip_vs_wrr, ip_vs_sh
modprobe -a softdog br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh

cat <<EOF | sudo tee /etc/modules-load.d/20-kubegg-modules.conf
softdog
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

systemctl enable systemd-modules-load
```

## 内核参数调优

```bash
# create tuned profile dir
mkdir -p /etc/tuned/{oltp,olap,crit,tiny}
```

### OLTP 调优

常规OLTP模板，优化延迟（默认值）

```bash
# node_mem_bytes
echo $(expr $(getconf _PHYS_PAGES) / 1 '*' $(getconf PAGE_SIZE))
```

```bash
# render tuned profiles
cat <<EOF | sudo tee /etc/tuned/oltp/tuned.conf
[main]
summary=Optimize for OLTP System
include=network-latency

[cpu]
force_latency=1
governor=performance
energy_perf_bias=performance
min_perf_pct=100

[vm]
# disable transparent hugepages
transparent_hugepages=never

[sysctl]
#-------------------------------------------------------------#
#                           KERNEL                            #
#-------------------------------------------------------------#
# disable numa balancing
kernel.numa_balancing=0

# total shmem size in bytes： \$(expr \$(getconf _PHYS_PAGES) /  * 0.75  \* \$(getconf PAGE_SIZE))
# total mem: {{ node_mem_bytes }}
kernel.shmall = 1373481984

# total shmax size in pages:  \$(expr \$(getconf _PHYS_PAGES) * 0.75 )
kernel.shmax = 335323

# total shmem segs 4096 -> 8192
kernel.shmmni=8192

# total msg queue number, set to mem size in MB
kernel.msgmni=32768

# max length of message queue
kernel.msgmnb=65536

# max size of message
kernel.msgmax=65536

kernel.pid_max=131072

# max(Sem in Set)=2048, max(Sem)=max(Sem in Set) x max(SemSet) , max(Sem per Ops)=2048, max(SemSet)=65536
kernel.sem=2048 134217728 2048 65536

# do not sched postgres process in group
kernel.sched_autogroup_enabled = 0

# total time the scheduler will consider a migrated process cache hot and, thus, less likely to be remigrated
# defaut = 0.5ms (500000ns), update to 5ms , depending on your typical query (e.g < 1ms)
kernel.sched_migration_cost_ns=5000000

#-------------------------------------------------------------#
#                             VM                              #
#-------------------------------------------------------------#
# try not using swap
vm.swappiness=1

# disable when most mem are for file cache
vm.zone_reclaim_mode=0

vm.overcommit_memory=0
vm.overcommit_ratio=100

# vm.dirty_background_bytes=67108864 # 64MB mem (2xRAID cache) wake the bgwriter
vm.dirty_background_ratio=3       # latency-performance default
vm.dirty_ratio=30                 # latency-performance default

# deny access on 0x00000 - 0x10000
vm.mmap_min_addr=65536


#-------------------------------------------------------------#
#                        Filesystem                           #
#-------------------------------------------------------------#
# max open files: 382589 -> 167772160
fs.file-max=167772160

# max concurrent unfinished async io, should be larger than 1M.  65536->1M
fs.aio-max-nr=1048576


#-------------------------------------------------------------#
#                          Network                            #
#-------------------------------------------------------------#
# max connection in listen queue (triggers retrans if full)
net.core.somaxconn=65535
net.core.netdev_max_backlog=8192
# tcp receive/transmit buffer default = 256KiB
net.core.rmem_default=262144
net.core.wmem_default=262144
# receive/transmit buffer limit = 4MiB
net.core.rmem_max=4194304
net.core.wmem_max=4194304

# ip options
net.ipv4.ip_forward=1
net.ipv4.ip_nonlocal_bind=1
net.ipv4.ip_local_port_range=32768 65000

# tcp options
net.ipv4.tcp_timestamps=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_tw_recycle=0
net.ipv4.tcp_syncookies=0
net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_syn_retries=1

# tcp read/write buffer
net.ipv4.tcp_rmem="4096 87380 16777216"
net.ipv4.tcp_wmem="4096 16384 16777216"
net.ipv4.udp_mem="3145728 4194304 16777216"

# tcp probe fail interval: 75s -> 20s
net.ipv4.tcp_keepalive_intvl=20
# tcp break after 3 * 20s = 1m
net.ipv4.tcp_keepalive_probes=3
# probe peroid = 1 min
net.ipv4.tcp_keepalive_time=60

net.ipv4.tcp_fin_timeout=5
net.ipv4.tcp_max_tw_buckets=262144
net.ipv4.tcp_max_syn_backlog=8192
net.ipv4.neigh.default.gc_thresh1=80000
net.ipv4.neigh.default.gc_thresh2=90000
net.ipv4.neigh.default.gc_thresh3=100000

net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-arptables=1

# max connection tracking number
net.netfilter.nf_conntrack_max=1048576

EOF
```

### OLAP 调优

常规OLAP模板，优化吞吐量

```bash
# render tuned profiles
cat <<EOF | sudo tee /etc/tuned/olap/tuned.conf
[main]
summary=Optimize for OLAP System
include=network-throughput

[cpu]
force_latency=1
governor=performance
energy_perf_bias=performance
min_perf_pct=100

[vm]
# disable transparent hugepages
transparent_hugepages=never

[sysctl]
#-------------------------------------------------------------#
#                           KERNEL                            #
#-------------------------------------------------------------#
# disable numa balancing
kernel.numa_balancing=0

# total shmem size in bytes： \$(expr \$(getconf _PHYS_PAGES) /  * 0.75  \* \$(getconf PAGE_SIZE))
# total mem: {{ node_mem_bytes }}
kernel.shmall = 1373481984

# total shmax size in pages:  \$(expr \$(getconf _PHYS_PAGES) * 0.75 )
kernel.shmax = 335323

# total shmem segs 4096 -> 8192
kernel.shmmni=8192

# total msg queue number, set to mem size in MB
kernel.msgmni=32768

# max length of message queue
kernel.msgmnb=65536

# max size of message
kernel.msgmax=65536

kernel.pid_max=131072

# max(Sem in Set)=2048, max(Sem)=max(Sem in Set) x max(SemSet) , max(Sem per Ops)=2048, max(SemSet)=65536
kernel.sem=2048 134217728 2048 65536

# do not sched postgres process in group
kernel.sched_autogroup_enabled = 0

# total time the scheduler will consider a migrated process cache hot and, thus, less likely to be remigrated
# defaut = 0.5ms (500000ns), update to 5ms , depending on your typical query (e.g < 1ms)
kernel.sched_migration_cost_ns=5000000

#-------------------------------------------------------------#
#                             VM                              #
#-------------------------------------------------------------#
# try not using swap
vm.swappiness=1

# disable when most mem are for file cache
vm.zone_reclaim_mode=0

#vm.overcommit_memory=0
#vm.overcommit_ratio=100

# allow 90% dirty ratio on OLAP instance
vm.dirty_background_ratio = 20    # throughput-performance default
vm.dirty_ratio=90                 # throughput-performance default 40 -> 90

# deny access on 0x00000 - 0x10000
vm.mmap_min_addr=65536


#-------------------------------------------------------------#
#                        Filesystem                           #
#-------------------------------------------------------------#
# max open files: 382589 -> 167772160
fs.file-max=167772160

# max concurrent unfinished async io, should be larger than 1M.  65536->1M
fs.aio-max-nr=1048576


#-------------------------------------------------------------#
#                          Network                            #
#-------------------------------------------------------------#
# max connection in listen queue (triggers retrans if full)
net.core.somaxconn=65535
net.core.netdev_max_backlog=8192
# tcp receive/transmit buffer default = 256KiB
net.core.rmem_default=262144
net.core.wmem_default=262144
# receive/transmit buffer limit = 4MiB
net.core.rmem_max=4194304
net.core.wmem_max=4194304

# ip options
net.ipv4.ip_forward=1
net.ipv4.ip_nonlocal_bind=1
net.ipv4.ip_local_port_range=32768 65000

# tcp options
net.ipv4.tcp_timestamps=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_tw_recycle=0
net.ipv4.tcp_syncookies=0
net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_syn_retries=1

# tcp read/write buffer
net.ipv4.tcp_rmem="4096 87380 16777216"
net.ipv4.tcp_wmem="4096 16384 16777216"
net.ipv4.udp_mem="3145728 4194304 16777216"

# tcp probe fail interval: 75s -> 20s
net.ipv4.tcp_keepalive_intvl=20
# tcp break after 3 * 20s = 1m
net.ipv4.tcp_keepalive_probes=3
# probe peroid = 1 min
net.ipv4.tcp_keepalive_time=60

net.ipv4.tcp_fin_timeout=5
net.ipv4.tcp_max_tw_buckets=262144
net.ipv4.tcp_max_syn_backlog=8192
net.ipv4.neigh.default.gc_thresh1=80000
net.ipv4.neigh.default.gc_thresh2=90000
net.ipv4.neigh.default.gc_thresh3=100000

net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-arptables=1

# max connection tracking number
net.netfilter.nf_conntrack_max=1048576

EOF
```

### CRIT 调优

核心金融业务模板，优化脏页数量

```bash
# render tuned profiles
cat <<EOF | sudo tee /etc/tuned/crit/tuned.conf
[main]
summary=Optimize for CRIT System
include=network-latency

[cpu]
force_latency=1
governor=performance
energy_perf_bias=performance
min_perf_pct=100

[vm]
# disable transparent hugepages
transparent_hugepages=never

[sysctl]
#-------------------------------------------------------------#
#                           KERNEL                            #
#-------------------------------------------------------------#
# disable numa balancing
kernel.numa_balancing=0

# total shmem size in bytes： \$(expr \$(getconf _PHYS_PAGES) /  * 0.75  \* \$(getconf PAGE_SIZE))
# total mem: {{ node_mem_bytes }}
kernel.shmall = 1373481984

# total shmax size in pages:  \$(expr \$(getconf _PHYS_PAGES) * 0.75 )
kernel.shmax = 335323

# total shmem segs 4096 -> 8192
kernel.shmmni=8192

# total msg queue number, set to mem size in MB
kernel.msgmni=32768

# max length of message queue
kernel.msgmnb=65536

# max size of message
kernel.msgmax=65536

kernel.pid_max=131072

# max(Sem in Set)=2048, max(Sem)=max(Sem in Set) x max(SemSet) , max(Sem per Ops)=2048, max(SemSet)=65536
kernel.sem=2048 134217728 2048 65536

# do not sched postgres process in group
kernel.sched_autogroup_enabled = 0

# total time the scheduler will consider a migrated process cache hot and, thus, less likely to be remigrated
# defaut = 0.5ms (500000ns), update to 5ms , depending on your typical query (e.g < 1ms)
kernel.sched_migration_cost_ns=5000000


#-------------------------------------------------------------#
#                             VM                              #
#-------------------------------------------------------------#
# try not using swap
vm.swappiness=1

# disable when most mem are for file cache
vm.zone_reclaim_mode=0

# 64MB mem (2xRAID cache) wake the bgwriter
vm.dirty_background_bytes=67108864
# vm.dirty_background_ratio=3       # latency-performance default
vm.dirty_ratio=10                   # latency-performance default

# deny access on 0x00000 - 0x10000
vm.mmap_min_addr=65536

#-------------------------------------------------------------#
#                        Filesystem                           #
#-------------------------------------------------------------#
# max open files: 382589 -> 167772160
fs.file-max=167772160

# max concurrent unfinished async io, should be larger than 1M.  65536->1M
fs.aio-max-nr=1048576


#-------------------------------------------------------------#
#                          Network                            #
#-------------------------------------------------------------#
# max connection in listen queue (triggers retrans if full)
net.core.somaxconn=65535
net.core.netdev_max_backlog=8192
# tcp receive/transmit buffer default = 256KiB
net.core.rmem_default=262144
net.core.wmem_default=262144
# receive/transmit buffer limit = 4MiB
net.core.rmem_max=4194304
net.core.wmem_max=4194304

# ip options
net.ipv4.ip_forward=1
net.ipv4.ip_nonlocal_bind=1
net.ipv4.ip_local_port_range=32768 65000

# tcp options
net.ipv4.tcp_timestamps=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_tw_recycle=0
net.ipv4.tcp_syncookies=0
net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_syn_retries=1

# tcp read/write buffer
net.ipv4.tcp_rmem="4096 87380 16777216"
net.ipv4.tcp_wmem="4096 16384 16777216"
net.ipv4.udp_mem="3145728 4194304 16777216"

# tcp probe fail interval: 75s -> 20s
net.ipv4.tcp_keepalive_intvl=20
# tcp break after 3 * 20s = 1m
net.ipv4.tcp_keepalive_probes=3
# probe peroid = 1 min
net.ipv4.tcp_keepalive_time=60

net.ipv4.tcp_fin_timeout=5
net.ipv4.tcp_max_tw_buckets=262144
net.ipv4.tcp_max_syn_backlog=8192
net.ipv4.neigh.default.gc_thresh1=80000
net.ipv4.neigh.default.gc_thresh2=90000
net.ipv4.neigh.default.gc_thresh3=100000

net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-arptables=1

# max connection tracking number
net.netfilter.nf_conntrack_max=1048576
EOF
```

### TINY 调优

微型虚拟机

```bash
# render tuned profiles
cat <<EOF | sudo tee /etc/tuned/tiny/tuned.conf
[main]
summary=Optimize for PostgreSQL TINY System
# include=virtual-guest

[vm]
# disable transparent hugepages
transparent_hugepages=never

[sysctl]
#-------------------------------------------------------------#
#                           KERNEL                            #
#-------------------------------------------------------------#
# disable numa balancing
kernel.numa_balancing=0

# total shmem size in bytes： \$(expr \$(getconf _PHYS_PAGES) /  * 0.75  \* \$(getconf PAGE_SIZE))
# total mem: {{ node_mem_bytes }}
kernel.shmall = 1373481984

# total shmax size in pages:  \$(expr \$(getconf _PHYS_PAGES) * 0.75 )
kernel.shmax = 335323

# If a workload mostly uses anonymous memory and it hits this limit, the entire
# working set is buffered for I/O, and any more write buffering would require
# swapping, so it's time to throttle writes until I/O can catch up.  Workloads
# that mostly use file mappings may be able to use even higher values.
#
# The generator of dirty data starts writeback at this percentage (system default
# is 20%)
vm.dirty_ratio = 50

# Filesystem I/O is usually much more efficient than swapping, so try to keep
# swapping low.  It's usually safe to go even lower than this on systems with
# server-grade storage.
vm.swappiness = 10


#-------------------------------------------------------------#
#                             VM                              #
#-------------------------------------------------------------#
#vm.overcommit_memory=0
#vm.overcommit_ratio=100

#-------------------------------------------------------------#
#                          Network                            #
#-------------------------------------------------------------#
# tcp options
net.ipv4.tcp_timestamps=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_tw_recycle=0
net.ipv4.tcp_syncookies=0
net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_syn_retries=1

# tcp probe fail interval: 75s -> 20s
net.ipv4.tcp_keepalive_intvl=20
# tcp break after 3 * 20s = 1m
net.ipv4.tcp_keepalive_probes=3
# probe peroid = 1 min
net.ipv4.tcp_keepalive_time=60
EOF
```

### 大页参数

```bash
cat <<EOF | sudo tee /etc/sysctl.d/hugepage.conf
vm.nr_hugepages = 0
EOF
```

### 激活 tuned 配置

```bash
tuned-adm profile oltp
sysctl -p /etc/sysctl.d/hugepage.conf
```

## ulimit 调优

``` bash
cat <<EOF | sudo tee /etc/security/limits.d/limits.conf
*    soft    nproc       1048576
*    hard    nproc       1048576
*    hard    nofile      1048576
*    soft    nofile      1048576
*    soft    stack       unlimited
*    hard    stack       unlimited
*    soft    core        unlimited
*    hard    core        unlimited
*    soft    memlock     2500000
*    hard    memlock     2500000

root soft    nproc       1048576
root hard    nproc       1048576
root hard    nofile      1048576
root soft    nofile      1048576
root soft    stack       unlimited
root hard    stack       unlimited
root soft    core        unlimited
root hard    core        unlimited
root soft    memlock     2500000
root hard    memlock     2500000

EOF
```

## 时间同步

### chrony 配置

```bash
# 设置时区
sudo timedatectl set-timezone Asia/Shanghai
```

```bash
# chrony 配置
cat <<EOF | sudo tee /etc/chrony.conf
# Use public servers from the pool.ntp.org project.
# pool cn.pool.ntp.org iburst
pool 10.1.1.11 iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 10 120

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
allow 127.0.0.1/32
allow 192.168.0.0/16
allow 10.0.0.0/8
allow ::/0

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking
EOF

systemctl enable chronyd
systemctl restart chronyd
```

### 计划任务

```bash
# 00 01 * * 1 postgres /pg/bin/pg-backup full
# 00 01 * * 2,3,4,5,6,7 postgres /pg/bin/pg-backup
# 后续添加
# sed -i '$a\00 01 * * 1 postgres /pg/bin/pg-backup full' /etc/crontab
```

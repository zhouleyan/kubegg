# Kubekey 源码分析

## 依赖包安装

相关源码路径：

- cmd/kk/pkg/pipelines/create_cluster.go
- cmd/kk/pkg/bootstrap/os/modules.go

### Get OS Data

Get OS release

```bash
cat /etc/os-release
```

### Sync

Sync repository iso file to all nodes

```shell
if [-d /tmp/kubekey];
then rm -rf /tmp/kubekey ;
fi

mkdir -m 777 -p /tmp/kubekey
```

```bash
scp id-verison-arch.iso root@xx.xx.xx.xx:/tmp/kubekey
```

### Mount

Mount iso file

```bash
sudo mount -t iso9660 -o loop path mountPath
```

### New repository client

```bash
which apt
which yum
```

### Backup

Backup original repository

```bash
# Debian
mv /etc/apt/sources.list /etc/apt/sources.list.kubekey.bak
mv /etc/apt/sources.list.d /etc/apt/sources.list.d.kubekey.bak
mkdir -p /etc/apt/sources.list.d

# Redhat
mv /etc/yum.repos.d /etc/yum.repos.d.kubekey.bak
mkdir -p /etc/yum.repos.d

```

### Add local repository

```bash
# Debian
rm -rf /etc/apt/sources.list.d/*

echo 'deb [trusted=yes]  file:///tmp/kubekey/iso   /' > /etc/apt/sources.list.d/kubekey.list

sudo apt-get update

# Redhat
rm -rf /etc/yum.repos.d/*

cat <<EOF> /etc/yum.repos.d/CentOS-local.repo
[base-local]
name=rpms-local

baseurl=file:///tmp/kubekey/iso

enabled=1

gpgcheck=0

EOF

yum clean all && yum makecache
```

### Install Packages

```bash

# Debian
sudo apt-get update
apt install -y socat conntrack ipset ebtables chrony ipvsadm

# Redhat
yum clean all && yum makecache
yum install -y openssl socat conntrack ipset ebtables chrony ipvsadm
```

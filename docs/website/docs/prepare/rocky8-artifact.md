# Rocky8 离线包

## Linux 主机准备

基于上一步初始化的 Rocky8 主机，克隆一个 artifact 主机用于制作离线包

| 系统            | 最低要求            |
| --------------- | ------------------- |
| Rocky Linux 8.9 | CPU: 2core, RAM: 4G |

## 本地 Yum 源制作

### 安装工具

```bash
dnf install -q -y dnf-plugins-core createrepo modulemd-tools mkisofs epel-release \
&& dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo \
&& dnf makecache

dnf update
```

### 添加 MySQL 源

```bash
wget https://dev.mysql.com/get/mysql84-community-release-el8-1.noarch.rpm

dnf install ./mysql84-community-release-el8-1.noarch.rpm

# 查看 MySQL Yum 源是否成功安装
dnf repolist enabled | grep "mysql.*-community.*"

dnf repolist all | grep mysql

# 禁用 MySQL 8.4
dnf config-manager --disable mysql-8.4-lts-community && \
dnf config-manager --disable mysql-tools-8.4-lts-community && \
dnf config-manager --enable powertools && \
dnf config-manager --enable mysql80-community && \
dnf config-manager --enable mysql-tools-community

# 或者手动修改 /etc/yum.repos.d/mysql-community.repo 文件
# 设置 MySQL 8.0 记录的 enabled=1，其他版本 enabled=0

dnf repolist enabled | grep mysql

# 禁用 E EL8 默认的 MySQL Module，保证 MySQL 安装来自 MySQL 的 Repo
dnf module disable mysql
```

### 创建 Repo

```bash
dnf download --resolve --alldeps --downloaddir=el-8.9-x86_64-rpms vim sudo curl wget bind-utils lz4 bash-completion net-tools tcpdump tree telnet openssl tar nss nss-sysinit nss-tools chrony mlocate sysstat iputils psmisc rsync libseccomp ebtables iptables ethtool nfs-utils glusterfs-client jq conntrack conntrack-tools socat ipset ipvsadm yum-utils mysql-community-server

createrepo --update -d ./el-8.9-x86_64-rpms
cd ./el-8.9-x86_64-rpms
repo2module ./
createrepo_mod ./
cd ..
mkisofs -r -o ./el-8.9-x86_64-rpms.iso ./el-8.9-x86_64-rpms
```

# 创建本地 Yum 源

## 导入离线包

```bash
mkdir -p /tmp/kubegg
mkdir -p /www/kubegg/iso

# 从 artifact 主机上传 iso 文件
# /tmp/kubegg/el-8.10-x86_64-rpms.iso
scp el-8.10-x86_64-rpms.iso root@10.1.1.11:/tmp/kubegg
```

挂载 iso 文件

```bash
mount -t iso9660 -o loop /tmp/kubegg/el-8.10-x86_64-rpms.iso /www/kubegg/iso
# 开机自动挂载
sed -i '$a\/root\/el-8.10-x86_64-rpms.iso \/www\/kubegg\/iso       iso9660 loop,defaults   0 0\n' /etc/fstab
```

## 新建本地源

在其他节点上添加本地源，确保节点网络联通，并可解析 `h.kubegg.local` DNS

```bash
# 备份原始源
mv /etc/yum.repos.d /etc/yum.repos.d.kubegg.bak && \
mkdir -p /etc/yum.repos.d

cat <<EOF | tee /etc/yum.repos.d/kubegg-local.repo
[kubegg-local]
name=Kubegg Local Yum Repo kubegg

baseurl=https://h.kubegg.local/kubegg/iso

skip_if_unavailable = 1

enabled=1

priority = 1

module_hotfixes=1

gpgcheck=0

EOF
```

```bash
yum clean all && yum makecache
```

## 安装包

节点即可通过 `infra` 节点本地源安装包

```bash
# 例如 PostgreSQL
yum install postgresql16*
```

## 还原默认源

安装完成后，还原节点默认 Yum 源

```bash
rm -rf /etc/yum.repos.d && \
mv /etc/yum.repos.d.kubegg.bak /etc/yum.repos.d && \
yum clean all && yum makecache
```

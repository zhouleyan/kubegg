# Rocky Linux 9.4 系统初始化

## 网络配置

```bash
# 重新生成 machine-id, UUID
rm -rf /etc/machine-id && \
systemd-machine-id-setup

# 查看当前网卡列表与 UUID
nmcli con show
# 删除要更改 UUID 的网络连接
nmcli con delete uuid 87ca36bf-f2ad-3608-99a1-11c1a7b5d8f4
# 重新生成 UUID
nmcli con add type ethernet ifname ens160 con-name ens160
# 重新启动网络连接
nmcli con up ens160

# 修改静态 IPv4地址
nmcli con mod ens160 ipv4.addresses 10.1.1.14/24
nmcli con mod ens160 ipv4.gateway 10.1.1.2
nmcli con mod ens160 ipv4.method manual
nmcli con mod ens160 ipv4.dns "10.1.1.2,8.8.8.8,8.8.4.4"
nmcli con up ens160
```

## 磁盘挂载

挂在两个硬盘

`/dev/sda`: `20G` `/`

`/dev/sdb`: `20G`

### 创建 PV

```bash
# pvcreare - Initialize physical volume(s) for use by LVM
# This command registers the partition sdb as physical volume to LVM
pvcreate /dev/sdb
pvcreate /dev/sdc

# The following command can be used to get all currently registered physical volumes
sudo pvs
```

### 创建 VG

```bash
vgcreate rl /dev/sdb
```

### 加入 VG

```bash
vgextend rl /dev/sdc
```

### 查看 VG

```bash
vgdisplay rl
```

### 创建 LV

```bash
# 当前 vg 总容量 60GiB
# 创建两个各 30GiB lv rl-home rl-data
lvcreate -n home -L 30G rl
lvcreate -n data -l 100%FREE rl

lvs
lvdisplay
lsblk -f
```

### 格式化 LV

```bash
mkfs.xfs /dev/mapper/rl-home
mkfs.xfs /dev/mapper/rl-data
```

### 挂载 LV

```bash
mkdir -p /data
mount /dev/mapper/rl-home /home
mount /dev/mapper/rl-data /data

sed -i '$a\/dev/mapper/rl-home     /home                   xfs     defaults        0 0' /etc/fstab
sed -i '$a\/dev/mapper/rl-data     /data                   xfs     defaults        0 0' /etc/fstab

systemctl daemon-reload

du -sh /home
df -h /home
du -sh /data
df -h /data
```

### LV 扩容

```bash
# 向 VG 添加新的磁盘
pvcreate /dev/sdd
vgextend rl /dev/sdd
vgs
vgdisplay rl

# 扩容 /home

## 先扩 5G
lvextend -L 35G /dev/mapper/rl-home

## 再扩剩余的 100%
lvextend -l +100%FREE /dev/mapper/rl-home

## 扩容文件系统
## xfs 使用 xfs_growfs
## 其他文件系统使用 resize2fs
xfs_growfs /dev/mapper/rl-home

# 查看扩容后结果
df -h /dev/mapper/rl-home
```

### LV 缩容

```bash
# XFS 不支持在线缩容
```

### LV 卷快照

```bash
# 创建一个逻辑卷，用于存放 home 的快照，容量必须一样（确保 VG 有足够容量）
lvcreate -L 29.99G -s -n SNAP /dev/mapper/rl-data

# 先删除数据
rm -rf /data/*
umount /dev/mapper/rl-data

# 恢复快照
lvconvert --merge /dev/rl/SNAP
mount /dev/mapper/rl-data
```

### LV 卸载/删除

```bash
rm -rf /home/*
umount /dev/mapper/rl-home
sed -i '/^[^#]*rl-home.*/s/^/\# /g' /etc/fstab
lvremove /dev/mapper/rl-home

rm -rf /data/*
umount /dev/mapper/rl-data
sed -i '/^[^#]*rl-data.*/s/^/\# /g' /etc/fstab
lvremove /dev/mapper/rl-data
lvs
```

### VG 删除

```bash
vgremove rl
```

### PV 删除

```bash
pvremove /dev/sdb
pvremove /dev/sdc
pvremove /dev/sdd
```

### 删除用户/组

```bash
userdel dba
rm -rf /etc/sudoers.d/dba
cat /etc/passwd

groupdel admin
cat /etc/group
```

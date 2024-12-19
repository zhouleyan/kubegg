# 其他

Linux 备忘

## 常用命令

### 查看开机自启动服务

```bash
systemctl list-unit-files
```

### awk 实现不打乱顺序去重

```bash
# $$ 代表当前进程 id
tmpfile="$$.tmp"
awk '!x[$0]++{print >"'$tmpfile'"}' /etc/sysctl.conf
mv $tmpfile /etc/sysctl.conf
```

### vim paste

```bash
:set paste
```

### 锁定文件

```bash
# 锁定
chattr +i /etc

# 解锁
chattr -i /etc
```

### 通过 Publickey 进行 SSH 登录

本机 mac 访问 192.168.3.4

```bash
# 生成 ssh key
ssh-keygen -t ed25519 -C "zhouleyan's Macbook"

# 上传到目标 host
ssh-copy-id -i ~/.ssh/id_ed25519 root@192.168.3.4

# 创建普通用户
useradd -m zhouleyan
passwd zhouleyan # Zly123456!
```

### tcpdump 抓包

```bash
tcpdump -i lo -c 100 -pne tcp -w lo.cap
```

### 配置 github SSH

```bash
cat <<EOF | sudo tee ~/.ssh/config
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
EOF
```

### SSH 跳转

```bash
ssh -J zhouleyan@192.168.2.105 root@172.16.0.130
ssh -J zhouleyan@frp-oak.top:59417 root@172.16.0.130
```

### Rocky Linux 8.10 删除旧内核

```bash
dnf remove --oldinstallonly kernel

# 升级 rescue mode kernel
rm -f /boot/vmlinuz-0-rescue-* /boot/initramfs-0-rescue-*.img
/usr/lib/kernel/install.d/51-dracut-rescue.install add $(uname -r) "" /lib/modules/$(uname -r)/vmlinuz
```

解决升级内核后包依赖问题

```bash
# package grub2-tools-minimal-1:2.02-158.el8_10.rocky.0.1.x86_64 from baseos is filtered out by modular filtering
dnf --setopt=baseos.module_hotfixes=true install grub2-common grub2-pc grub2-pc-modules grub2-tools grub2-tools-extra grub2-tools-minimal
```

### "|| :" 作用

```bash
# command1
useradd -M -c 'MySQL user' -s /sbin/nologin -r mysql || :

# 加入 command1 执行失败，执行”:“，相当于什么都没做，但是返回码为 “0”，不影响后续命令执行
# 例如已经存在 mysql，执行 useradd -M -c 'MySQL user' -s /sbin/nologin -r mysql || : && echo 'success'
```

### Systemd

所有的启动设置之前，都可以加上一个连词号（-），表示 "抑制错误"，即发生错误的时候，不影响其他命令的执行。比如，EnvironmentFile=-/etc/sysconfig/sshd（注意等号后面的那个连词号），就表示即使 / etc/sysconfig/sshd 文件不存在，也不会抛出错误。

### Linux 命令返回码

在 Linux 中，可以通过特殊变量$?来获取上一个命令的返回码。

```bash
ls /nonexistent_directory
echo "Last command return code: $?"
```

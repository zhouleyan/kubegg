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

### Rocky Linux 8.9 删除旧内核

```bash
dnf remove --oldinstallonly kernel
```

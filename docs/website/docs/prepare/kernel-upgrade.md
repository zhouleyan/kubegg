# 内核升级

debian12 内核由 `6.1.0-13` 升级到 `6.5.0-0`。

## 内核按需升级（可选）

### 添加 backports 源

```bash
# bookworm-backports 已包含在国内源里
# echo "deb http://deb.debian.org/debian bookworm-backports main" > /etc/apt/sources.list.d/backports.list

apt update
apt install -t bookworm-backports linux-image-amd64
apt install -t bookworm-backports linux-headers-amd64

update-grub
```

### 重启服务器

```bash
shutdown -r now
```

### 查看内核版本

```bash
uname -r
```

### 卸载旧内核

```bash
dpkg --list | grep linux-image
# 卸载制定的旧内核
apt purge linux-image-6.1.0-13-amd64
```

### 删除旧头文件

```bash
dpkg --list | grep linux-headers
apt purge linux-headers-6.1.0-13-amd64
```

如果使用中的内核缺失头文件，可以使用如下命令安装

```bash
apt install linux-headers-$(uname -r)
```

### 保持内核最新（可选）

把如下命令添加到 `/etc/apt/preferences.d/pinning.pref` 文件中（如果没有，可创建）

```bash
Package: linux-image-amd64 linux-headers-amd64
Pin: release n=buster-backports
Pin-Priority: 900
```

## 内核回退

内核版本回退至 `6.1.0-13`

### 查找对应的版本内核

```bash
apt-cache search linux-image | grep 6.1.0-13-amd64
```

### 安装指定的版本内核的 Linux 头文件与镜像

```bash
apt install linux-image-6.1.0-13-amd64 linux-headers-6.1.0-13-amd64
```

### 查看当前系统中内核的启动顺序

```bash
grep menuentry /boot/grub/grub.cfg
```

重启进入 `Advanced options for Debian GNU/Linux`

选择 `Debian GNU/Linux, with Linux 6.1.0-13-amd64` 进入操作系统

### 移除不用的内核

查询不包括当前内核版本的其它所有内核版本

```bash
dpkg -l | tail -n +6| grep -E 'linux-image-[0-9]+'| grep -Fv $(uname -r)

ii  linux-image-6.5.0-0.deb12.1-amd64    6.5.3-1~bpo12+1                amd64        Linux 6.5 for 64-bit PCs (signed)
```

输出的内容中可能会包括内核映像的如下三种状态：
rc：表示已经被移除
ii：表示符合移除条件（可移除）
iU：已进入 apt 安装队列，但还未被安装（不可移除）

我这里的 `6.5.0-0` 内核为可移除状态，可以执行如下命令删除

```bash
apt purge linux-image-6.5.0-0* linux-headers-6.5.0-0*
apt autoremove
```

重启验证。

```bash
shutdown -r now
```

## Install Linux Kernel 6.6 on Debian12

```bash
sudo apt update
```

```bash
sudo apt upgrade
```

```bash
sudo apt install lsb-release software-properties-common apt-transport-https ca-certificates curl -y
```

```bash
curl -fSsL https://pkgs.zabbly.com/key.asc | gpg --dearmor | sudo tee /usr/share/keyrings/linux-zabbly.gpg > /dev/null
```

```bash
codename=$(lsb_release -sc) && echo deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/linux-zabbly.gpg] https://pkgs.zabbly.com/kernel/stable $codename main | sudo tee /etc/apt/sources.list.d/linux-zabbly.list
```

```bash
sudo apt update
```

```bash
sudo apt install linux-zabbly
```

```bash
echo -e "# Used for shielding pcspkr module, this module seems to be related to buzzer.\nblacklist pcspkr" | sudo tee /etc/modprobe.d/blacklist-pcspkr.conf
```

```bash
sudo reboot
```

```bash
sudo reboot
```

```bash
dpkg --list | grep linux-image
# 卸指定的旧内核
apt purge linux-image-6.1.0-13-amd64
```

```bash
dpkg --list | grep linux-headers
apt purge linux-headers-6.1.0-13-amd64
```

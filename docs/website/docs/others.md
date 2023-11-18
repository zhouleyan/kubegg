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

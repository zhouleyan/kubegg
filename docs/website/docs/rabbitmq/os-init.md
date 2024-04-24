# OS 初始化

## Linux 主机准备

选择 Rocky Linux 8.9 操作系统

| 系统            | 最低要求            |
| --------------- | ------------------- |
| Rocky Linux 8.9 | CPU: 2core, RAM: 4G |

### 网络配置

```bash
# 生成 uuid
uuidgen

# 生成 e11ffc43-5f46-487a-94f5-73c603924e88，替换下面网络 UUID

vi /etc/sysconfig/network-scripts/ifcfg-ens160

# 修改静态 IP 等配置

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=eui64
NAME=ens160
UUID=3fede721-5bc0-4043-bfe6-9597424ddc79
DEVICE=ens160
ONBOOT=yes
IPADDR=10.1.1.10
PREFIX=24
GATEWAY=10.1.1.2
DNS1=10.1.1.2
DNS2=8.8.8.8
DNS3=8.8.4.4

```

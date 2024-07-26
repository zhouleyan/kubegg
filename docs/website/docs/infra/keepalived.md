# Keepalived 安装

## Linux 主机准备

基于上一步安装完 Infra 的主机，继续安装 Keepalived

## Keepalived 配置

### Keepalived config

```bash
cat <<EOF | tee /etc/keepalived/keepalived.conf
global_defs {
  router_id infra
   enable_script_security
   script_user root
}

vrrp_instance infra {

  state MASTER

  interface ens160

  virtual_router_id 128

  priority 128

  # preempt is enabled by default

  advert_int 1
  unicast_src_ip 10.1.1.11
  unicast_peer {
  }

  authentication {
    auth_type PASS
    auth_pass infra-128
  }

  virtual_ipaddress {
    10.1.1.99
  }

}

EOF
```

## Keepalived 启动

```bash
systemctl enable keepalived && \
systemctl restart keepalived
```

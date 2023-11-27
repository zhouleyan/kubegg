# DNS 服务器配置

## 环境信息

选择 Debian 操作系统

| 系统        | 最低要求            |
| ----------- | ------------------- |
| Debian 12.3 | CPU: 1core,RAM: 1G |

ns1、ns2 作分别为 DNS 主从服务器

| Host     | Role       | Private FQDN          | Private IP Address |
| -------- | ---------- | --------------------- | ------------------ |
| ns1      | DNS 主节点 | ns1.kubegg.local      | 172.16.239.11      |
| ns2      | DNS 从节点 | ns2.kubegg.local      | 172.16.239.12      |
| registry | 镜像仓库   | registry.kubegg.local | 172.16.239.11      |
| -        | vip        | lb.kubegg.local       | 172.16.239.100     |
| master1  | master1    | master1               | 172.16.239.101     |
| master2  | master2    | master2               | 172.16.239.102     |
| node1    | node1      | node1                 | 172.16.239.103     |
| node2    | node1      | node2                 | 172.16.239.104     |
| node3    | node1      | node3                 | 172.16.239.105     |

## 安装 DNS 服务

### 安装 BIND

On both DNS servers,ns1 and ns2,update the `apt` package cacahe by typing:

```bash
sudo apt update
```

Then install BIND on each maching:

```bash
sudo apt install bind9 bind9utils bind9-doc
```

### 配置 IPv4 模式

In this case,set BIND to IPv4 mode.On both servers,edit the `named` default settings file using your preferred text editor:

```bash
sudo nano /etc/default/named
```

Add `-4` to the end of the `OPTIONS` parameter:

```bash
...

OPTIONS="-u bind -4"
```

Restart BIND to implement the changes:

```bash
sudo systemctl restart bind9
```

## 配置 DNS 主节点

BIND's configuration consists of multiple files that are included from the main configuration file,`named.conf`.These file names begin with named because that is the name of the process that BIND runs(with `named` being short for "**name d**aemon",as in "domain name daemon").We will start with configuring the `named.conf.options` file.

### 配置 Options 文件

On `ns1`,open the named.conf.options file for editing:

```bash title="/etc/bind/named.conf.options"
vim /etc/bind/named.conf.options
```

Above the existing `options` block,create a new ACL(access control list) block called `trusted`.This is where you will define a list of clients from which you will allow recursive DNS queries (i.e. your servers that are in the same datacenter as `ns1`).Add the following lines to add `ns1`,`ns2` and the others to your list of trusted clients,being sure to replace the example privare IP addresses with those of your own servers:

```bash title="/etc/bind/named.conf.options"
vim /etc/bind/named.conf.options

acl "trusted" {
    172.16.239.11;  # ns1
    172.16.239.12;  # ns2
    # ...
    
};

# Or

acl internal-network {
    172.16.239.0/24;
};

options {
    directory "/var/cache/bind";

    forwarders {
        172.16.239.2;
        8.8.8.8;
        8.8.4.4;
    };

    allow-query { localhost; internal-network; };

    allow-transfer { localhost; };

    recursion yes;

    dnssec-validation auto;

    listen-on-v6 { any; };
};
```

Next,you will specify your DNS zones by configuring the `named.conf.internal-zones`.

### named 配置

On `ns1`,open the `named.conf` file and add:

```bash title="/etc/bind/named.conf"
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
# add
include "/etc/bind/named.conf.internal-zones";
```

Create the `named.conf.internal-zones` file for editing:

```bash title="/etc/bind/named.conf.internal-zones"
cat <<EOF | sudo tee /etc/bind/named.conf.internal-zones
zone "kubegg.local" {
    type primary;
    file "/etc/bind/zones/db.kubegg.local";  # zone file path
    allow-transfer { 172.16.239.12; };
};

zone "239.16.172.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.172.16.239";  # 172.16.239.0/24 subnet
    allow-transfer { 172.16.239.12; };
};
EOF
```

Now that your zones are specified in BIND, you need to create the corresponding forward and reverse zone files.

### 创建 Forward Zone 文件

The forward zone file is where you define DNS records for forward DNS lookups.That is,when the DNS receives a name query,`ns1.kubegg.local` for example,it will look in the forward zone file to resolve `ns1`'s corresponding private IP address.

```bash
mkdir -p /etc/bind/zones

vim /etc/bind/zones/db.kubegg.local
```

Initially,it will contain content like the following:

```bash title="/etc/bind/zones/db.kubegg.local"
cat <<EOF | sudo tee /etc/bind/zones/db.kubegg.local
\$TTL    604800
@       IN      SOA     ns1.kubegg.local. admin.kubegg.local. (
                        # any numerical values are OK for serial number
                        # recommended : [YYYYMMDDnn] (update date + number)
                     2023112401         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
; name servers - NS records
        IN      NS      ns1.kubegg.local.
        IN      NS      ns2.kubegg.local.

; name servers - A records
ns1.kubegg.local.         IN      A       172.16.239.11
ns2.kubegg.local.         IN      A       172.16.239.12

; 172.16.239.0/24 - A records
registry.kubegg.local. IN      A       172.16.239.11
lb.kubegg.local.       IN      A       172.16.239.100
EOF
```

### 创建 Reverse Zone 文件

### 验证 BIND 配置文件

### 重启 BIND

## 配置 DNS 从节点

## 配置 DNS 客户端

## 测试 DNS

## 维护 DNS 记录

``

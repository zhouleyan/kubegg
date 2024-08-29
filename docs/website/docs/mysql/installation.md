# MySQL 安装

## Linux 主机准备

基于上一步初始化的 Rocky8 主机，克隆一个主机用于 MySQL 安装

| 系统            | 最低要求            | MySQL 版本 |
| --------------- | ------------------- | ---------- |
| Rocky Linux 8.10 | CPU: 2core, RAM: 4G | 8.0.39     |

## 确定系统信息

```bash
cat /etc/os-release
```

## 创建 MySQL 用户

```bash
# mysql_dbsu: mysql                    # os dbsu name, mysql by default, better not change it
# mysql_dbsu_uid: 27                   # os dbsu uid and gid, 306 for default mysql users and groups
# mysql_dbsu_home: /var/lib/mysql      # mysql home directory, `/var/lib/mysql` by default
# mysql_data: /data/mysql              # mysql data directory, `/data/mysql` by default

# 创建用户组
groupadd -g 27 mysql
# 创建用户
useradd -u 27 -g mysql -G mysql -d /var/lib/mysql -s /bin/bash -m mysql
mkdir -p /var/lib/mysql/.ssh
ssh-keygen -t rsa -b 4096 -f /var/lib/mysql/.ssh/id_rsa -m pem -N ""
chown -R mysql:mysql /var/lib/mysql
```

## 导入离线包

```bash
mkdir -p /tmp/kubegg/iso

# 从 artifact 主机上传 iso 文件
# /tmp/kubegg/el-8.10-x86_64-rpms.iso
scp el-8.10-x86_64-rpms.iso root@10.1.1.11:/tmp/kubegg
```

挂载 iso 文件

```bash
mount -t iso9660 -o loop /tmp/kubegg/el-8.10-x86_64-rpms.iso /tmp/kubegg/iso
```

新建本地源

```bash
# 备份原始源
mv /etc/yum.repos.d /etc/yum.repos.d.kubegg.bak

mkdir -p /etc/yum.repos.d

# 添加本地源
cat <<EOF | tee /etc/yum.repos.d/kubegg-local.repo
[base-local]
name=rpms-local

baseurl=file:///tmp/kubegg/iso

enabled=1

gpgcheck=0

EOF
```

```bash
yum clean all && yum makecache
```

## 安装 MySQL

```bash
yum install mysql-community* mysqld_exporter
```

## 卸载 iso 文件

```bash
umount /tmp/kubegg/iso
```

## 重置源

```bash
rm -rf /etc/yum.repos.d
mv /etc/yum.repos.d.kubegg.bak /etc/yum.repos.d
```

```bash
yum clean all && yum makecache
```

## 创建 MySQL 文件夹

```bash
# /var/run/mysqld  0755
mkdir -p /var/run/mysqld && \
chown mysql:mysql /var/run/mysqld && \
chmod 0755 /var/run/mysqld

# /etc/my.cnf.d    0700
mkdir -p /etc/my.cnf.d && \
chown mysql:mysql /etc/my.cnf.d && \
chmod 0700 /etc/my.cnf.d

# /data/mysql/data 0700
mkdir -p /data/mysql/data && \
chown mysql:mysql /data/mysql/data && \
chmod 0700 /data/mysql/data

# /data/mysql/log  0700
mkdir -p /data/mysql/log && \
chown mysql:mysql /data/mysql/log && \
chmod 0700 /data/mysql/log

ln -s /data/mysql /my
```

## MySQL 配置

```bash
cat <<EOF | tee /etc/my.cnf
[mysqld]
datadir = /data/mysql/data
port = 3306
bind-address = '0.0.0.0'
socket = /var/lib/mysql/mysql.sock
pid-file = /var/run/mysqld/mysqld.pid

# replication
server-id = 0

# primary configuration
# log_bin = mysql-bin
# log-bin-index = mysql-bin.index
# max_binlog_size = 1G

# replica configuration
# read_only
# relay-log = relay-bin
# relay-log-index = relay-bin.index

# logging
general_log = 1
general_log_file = /data/mysql/log/mysql.log
log-error = /data/mysql/log/error.log
slow_query_log = 1
slow_query_log_file = /data/mysql/log/slow.log

# misc
skip-name-resolve

# parameters
max_connections = 100
key_buffer_size = 256M
max_allowed_packet = 64M
table_open_cache = 256
sort_buffer_size = 1M
read_buffer_size = 1M
read_rnd_buffer_size = 4M
myisam_sort_buffer_size = 64M
thread_cache_size = 8
tmp_table_size = 16M
max_heap_table_size = 16M
group_concat_max_len = 1024
join_buffer_size = 262144
lower_case_table_names = 0
wait_timeout = 28800
event_scheduler = 'OFF'
innodb_file_per_table = 1
innodb_buffer_pool_size = 256M
innodb_log_buffer_size = 8M
innodb_flush_log_at_trx_commit = 1
innodb_lock_wait_timeout = 50
long_query_time = 2

[client]
port = 3306
socket = /var/lib/mysql/mysql.sock
#password = <your_password>

[mysqldump]
quick
max_allowed_packet = 64MB

[mysqld_safe]
pid-file = /var/run/mysqld/mysqld.pid
#!includedir /etc/my.cnf.d

EOF
```

```bash
chown mysql:mysql /etc/my.cnf && \
chmod 0644 /etc/my.cnf
```

## MySQL 初始化

```bash
# mysqld 初始化
su -l mysql -c 'mysqld --initialize-insecure --datadir=/data/mysql/data'

systemctl restart mysqld
systemctl enable mysqld
systemctl daemon-reload

# 修改初始化密码
# 查看初始密码
# grep 'temporary password' /var/log/mysqld.log

rm -rf /root/.my.cnf
mysql -u root -NBe "ALTER USER root@localhost IDENTIFIED WITH mysql_native_password BY'DBUser.Root'; FLUSH PRIVILEGES;";

cat <<EOF | tee /root/.my.cnf
[client]
user = 'root'
password = 'DBUser.Root'
EOF

cat <<EOF | tee /var/lib/mysql/.my.cnf
[client]
user = 'root'
password = 'DBUser.Root'
EOF

chown -R mysql:mysql /var/lib/mysql/.my.cnf && \
chmod 0600 /var/lib/mysql/.my.cnf
```

## 验证登录

```bash
# 登录 MySQL
mysql -uroot -p

SHOW VARIABLES LIKE 'max_connections';

SHOW VARIABLES LIKE 'default%';
```

## mysqld_exporter 初始化

创建 mysqld_exporter 用户

```bash
mysql -NBe "CREATE USER IF NOT EXISTS'dbuser_monitor'@'localhost'IDENTIFIED BY'DBUser.Monitor'WITH MAX_USER_CONNECTIONS 3;";
mysql -NBe "GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO'dbuser_monitor'@'localhost';";
mysql -NBe "FLUSH PRIVILEGES;"
```

mysqld_exporter 配置

```bash
cat <<EOF | tee /etc/default/mysqld_exporter
MYSQLD_EXPORTER_OPTS="--mysqld.address=:3306 --mysqld.username='dbuser_monitor'--web.listen-address=:9104 --web.telemetry-path='/metrics'--collect.auto_increment.columns --collect.binlog_size --collect.engine_innodb_status --collect.engine_tokudb_status --collect.global_status --collect.global_variables --collect.heartbeat.utc --collect.info_schema.clientstats --collect.info_schema.innodb_metrics --collect.info_schema.innodb_tablespaces --collect.info_schema.innodb_cmp --collect.info_schema.innodb_cmpmem --collect.info_schema.processlist --collect.info_schema.query_response_time --collect.info_schema.replica_host --collect.info_schema.tables --collect.info_schema.tablestats --collect.info_schema.schemastats --collect.info_schema.userstats --collect.mysql.user --collect.perf_schema.eventsstatements --collect.perf_schema.eventsstatementssum --collect.perf_schema.eventswaits --collect.perf_schema.file_events --collect.perf_schema.file_instances --collect.perf_schema.indexiowaits --collect.perf_schema.memory_events --collect.perf_schema.tableiowaits --collect.perf_schema.tablelocks --collect.perf_schema.replication_group_members --collect.perf_schema.replication_group_member_stats --collect.perf_schema.replication_applier_status_by_worker --collect.slave_status --collect.slave_hosts --collect.sys.user_summary"
MYSQLD_EXPORTER_PASSWORD="DBUser.Monitor"
EOF

chown -R mysql:mysql /etc/default/mysqld_exporter && \
chmod 0600 /etc/default/mysqld_exporter
```

mysqld_exporter 启动

```bash
systemctl restart mysqld_exporter
systemctl enable mysqld_exporter
systemctl daemon-reload
```

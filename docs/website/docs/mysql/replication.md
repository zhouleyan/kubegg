# MySQL 主从配置

将上一篇部署配置的 MySQL 节点（10.1.1.20）作为主节点，添加一个 MySQL 从节点，配置主从架构

## MySQL 主节点配置

编辑 MySQL 配置文件 `/etc/my.cnf` ，添加以下配置

```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
max_binlog_size = 1G
```

重启 MySQL 服务使配置生效

```bash
sudo systemctl restart mysql
```

在 MySQL 命令行中，创建用于复制的用户并授予相应的权限

```sql
CREATE USER 'dbuser_replicator'@'10.1.1.21' IDENTIFIED WITH mysql_native_password BY 'DBUser.Replicator';
GRANT REPLICATION SLAVE ON *.* TO 'dbuser_replicator'@'10.1.1.21';
FLUSH PRIVILEGES;
```

获取主节点的 binlog 信息，用于从节点配置

```sql
SHOW MASTER STATUS;
```

```bash
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |   380332 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

记录 `File` 与 `Position` 的值

## 从节点配置

安装与主节点步骤相同，若直接克隆主节点，则需要删除以下文件保证从节点 MySQL uuid 与主节点不同

```bash
# 在从节点执行
# MySQL data 路径：/data/mysql/data
sudo rm -rf /data/mysql/data/auto.cnf

# 重启 MySQL 服务，重新生成 auto.cnf 更新 MySQL uuid
systemctl restart mysql
```

编辑 MySQL 配置文件 `/etc/my.cnf` ，添加以下配置

```ini
[mysqld]
server-id = 2

# replica configuration
read_only
relay-log = relay-bin
relay-log-index = relay-bin.index
```

重启 MySQL 服务使配置生效

```bash
sudo systemctl restart mysql
```

登录到 MySQL 服务器，配置从节点复制信息

```sql
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST = '10.1.1.20',
    SOURCE_USER = 'dbuser_replicator',
    SOURCE_PASSWORD = 'DBUser.Replicator',
    SOURCE_LOG_FILE = 'mysql-bin.000001',
    SOURCE_LOG_POS = 380332;
```

启动从节点复制进程，检查从节点复制状态

```sql
START REPLICA;
SHOW REPLICA STATUS\G
```

查看 `Replica_IO_Running`、`Replica_SQL_Running` 是否都为 `Yes` ，如果是，则表示复制正常

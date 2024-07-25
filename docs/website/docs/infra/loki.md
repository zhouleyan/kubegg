# Loki 安装

## Linux 主机准备

基于上一步安装完 Grafana 的主机，继续安装 Loki

## 创建 Loki 文件夹

```bash
mkdir -p /data/loki && \
chown -R loki:loki /data/loki && \
chmod 0700 /data/loki && \

mkdir -p /data/loki/data/rules/fake && \
chown loki:loki /data/loki/data/rules/fake && \
chmod 0700 /data/loki/data/rules/fake && \

mkdir -p /etc/loki && \
chown root /etc/loki && \
chmod 0755 /etc/loki
```

## Loki 配置

### Loki systemd service

```bash
mv /etc/systemd/system/loki.service /etc/systemd/system/loki.service.bk

cat <<EOF | tee /usr/lib/systemd/system/loki.service
# -*- mode: conf -*-

[Unit]
Description=The Loki Logging Service
Documentation=https://grafana.com/docs/loki/latest/
After=network.target

[Service]
User=loki
ExecStart=/usr/bin/loki -config.file /etc/loki/config.yml
ExecReload=/bin/kill -HUP \$MAINPID
TimeoutSec = 120
Restart = on-failure
RestartSec = 2
LimitNOFILE=16777216

[Install]
WantedBy=multi-user.target

EOF
```

### Loki config

```bash
cat <<EOF | tee /etc/loki/config.yml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /data/loki/chunks
      rules_directory: /data/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

ingester:
  wal:
    enabled: true
    dir: /data/loki/wal
    flush_on_shutdown: true     # Flush any ingested but not yet flushed chunks when loki shuts down
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 1h       # Any chunk not receiving new logs in this time will be flushed
  max_chunk_age: 1h           # All chunks will be flushed when they hit this age, default is 1h
  chunk_target_size: 1048576  # Loki will attempt to build chunks up to 1.5MB, flushing first if chunk_idle_period or max_chunk_age is reached first
  chunk_retain_period: 30s    # Must be greater than index read cache TTL if using an index cache (Default index read cache TTL is 5m)

# define data schema
schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

# increase resource limit
# https://grafana.com/docs/loki/latest/configuration/#limits_config
limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  split_queries_by_interval: 5m
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 64
  max_entries_limit_per_query: 50000
  max_chunks_per_query: 10000000
  max_query_series: 32768

# cache query results in memory
query_range:
  align_queries_with_step: true
  parallelise_shardable_queries: false
  max_retries: 3
  cache_results: true
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

storage_config:
  boltdb_shipper:
    active_index_directory: /data/loki/boltdb-shipper-active
    cache_location: /data/loki/boltdb-shipper-cache
    cache_ttl: 24h   # Can be increased for faster performance over longer query periods, uses more disk space

  tsdb_shipper:
    active_index_directory: /data/loki/tsdb-shipper-active
    cache_location: /data/loki/tsdb-shipper-cache
    cache_ttl: 24h   # Can be increased for faster performance over longer query periods, uses more disk space

# setup compactor working directory
compactor:
  working_directory: /data/loki/tmp/boltdb-shipper-compactor

# increase frontend limits
frontend:
  max_outstanding_per_tenant: 8192

# setup log data retention
table_manager:
  retention_deletes_enabled: true
  retention_period: 15d

# disable telemetry reporting
analytics:
  reporting_enabled: false

EOF
```

```bash
chown loki:loki /etc/loki/config.yml && \
chmod 0644 /etc/loki/config.yml
```

## Loki 启动

```bash
# port :3100
systemctl enable loki
systemctl restart loki
```

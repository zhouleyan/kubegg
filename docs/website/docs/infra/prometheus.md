# Prometheus 安装

## Linux 主机准备

基于上一步初始化的 HAProxy 主机，安装 Prometheus

| 系统            | 最低要求            | Prometheus 版本 |
| --------------- | ------------------- | --------------- |
| Rocky Linux 8.10 | CPU: 2core, RAM: 4G | prometheus2     |

## 创建 Prometheus 用户

```bash
# 创建 Infra 用户组
groupadd -r infra

# 把用户 prometheus 添加进 infra 组
usermod -aG infra prometheus
```

## 创建 Prometheus 文件夹

```bash
mkdir -p /data/prometheus && \
chown prometheus:prometheus /data/prometheus && \
chmod 0700 /data/prometheus

mkdir -p /data/prometheus/data && \
chown prometheus:prometheus /data/prometheus/data && \
chmod 0700 /data/prometheus/data

# prometheus_sd_dir: /etc/prometheus/targets
mkdir -p /etc/prometheus && \
chown prometheus:prometheus /etc/prometheus && \
chmod 0750 /etc/prometheus

mkdir -p /etc/prometheus/bin && \
chown prometheus:prometheus /etc/prometheus/bin && \
chmod 0750 /etc/prometheus/bin

mkdir -p /etc/prometheus/rules && \
chown prometheus:prometheus /etc/prometheus/rules && \
chmod 0750 /etc/prometheus/rules

mkdir -p /etc/prometheus/targets && \
chown prometheus:prometheus /etc/prometheus/targets && \
chmod 0750 /etc/prometheus/targets

mkdir -p /etc/prometheus/targets/node && \
chown prometheus:prometheus /etc/prometheus/targets/node && \
chmod 0750 /etc/prometheus/targets/node

mkdir -p /etc/prometheus/targets/ping && \
chown prometheus:prometheus /etc/prometheus/targets/ping && \
chmod 0750 /etc/prometheus/targets/ping

mkdir -p /etc/prometheus/targets/etcd && \
chown prometheus:prometheus /etc/prometheus/targets/etcd && \
chmod 0750 /etc/prometheus/targets/etcd

mkdir -p /etc/prometheus/targets/infra && \
chown prometheus:prometheus /etc/prometheus/targets/infra && \
chmod 0750 /etc/prometheus/targets/infra

mkdir -p /etc/prometheus/targets/mysql && \
chown prometheus:prometheus /etc/prometheus/targets/mysql && \
chmod 0750 /etc/prometheus/targets/mysql
```

## Prometheus 配置

### Prometheus systemd service

```bash
cat <<EOF | tee /usr/lib/systemd/system/prometheus.service
[Unit]
Description=The Prometheus monitoring system and time series database.
Documentation=https://prometheus.io
After=network.target

[Service]
EnvironmentFile=-/etc/default/prometheus
User=prometheus
ExecStart=/usr/bin/prometheus \$PROMETHEUS_OPTS
ExecReload=/bin/kill -HUP \$MAINPID
Restart=always
RestartSec=5s
LimitNOFILE=16777216

[Install]
WantedBy=multi-user.target

EOF
```

### Prometheus opts

```bash
cat <<EOF | tee /etc/default/prometheus
PROMETHEUS_OPTS='--config.file=/etc/prometheus/prometheus.yml --web.page-title="Kubegg Metrics"--storage.tsdb.path=/data/prometheus/data --storage.tsdb.retention.time=15d'

EOF
```

```bash
chown prometheus:prometheus /etc/default/prometheus && \
chmod 0755 /etc/default/prometheus
```

### Prometheus config

```yaml
cat <<EOF | tee /etc/prometheus/prometheus.yml
---
#--------------------------------------------------------------#
# Config FHS
#--------------------------------------------------------------#
# /etc/prometheus/
#  ^-----prometheus.yml    # prometheus main config file
#  ^-----alertmanager.yml  # alertmanger main config file
#  ^-----@bin              # util scripts: check,reload,status,new
#  ^-----@rules            # record & alerting rules definition
#
# {{prometheus_sd_dir}}
#            ^-----@node   # node static targets definition
#            ^-----@node   # ping targets definition
#            ^-----@pgsql  # pgsql static targets definition
#            ^-----@pgrds  # pgsql remote rds static targets
#            ^-----@redis  # redis static targets definition
#            ^-----@etcd   # etcd static targets definition
#            ^-----@minio  # minio static targets definition
#            ^-----@infra  # infra static targets definition
#            ^-----@mongo  # mongo static targets definition
#            ^-----@mysql  # mysql static targets definition
#--------------------------------------------------------------#


#--------------------------------------------------------------#
# Globals
#--------------------------------------------------------------#
global:
  scrape_interval: 10s
  evaluation_interval: 10s
  scrape_timeout: 8s


#--------------------------------------------------------------#
# Alerts
#--------------------------------------------------------------#
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['10.1.1.11:9093']
      scheme: http
      timeout: 10s
      api_version: v2


#--------------------------------------------------------------#
# Rules
#--------------------------------------------------------------#
rule_files:
  - rules/*.yml


#--------------------------------------------------------------#
# Targets Definition
#--------------------------------------------------------------#
# https://prometheus.io/docs/prometheus/latest/configuration/configuration/#file_sd_config
scrape_configs:


  #--------------------------------------------------------------#
  # job: prometheus
  # prometheus metrics
  #--------------------------------------------------------------#
  - job_name: prometheus
    metrics_path: /metrics
    honor_labels: true
    static_configs: [{targets: ['127.0.0.1:9090'] }]

  #--------------------------------------------------------------#
  # job: push
  # pushgateway metrics
  #--------------------------------------------------------------#
  - job_name: push
    metrics_path: /metrics
    honor_labels: true
    static_configs: [{targets: ['127.0.0.1:9091'] }]

  #--------------------------------------------------------------#
  # job: ping
  # blackbox exporter metrics
  #--------------------------------------------------------------#
  - job_name: ping
    metrics_path: /probe
    params: {module: [icmp]}
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/ping/*.yml]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115

  #--------------------------------------------------------------#
  # job: infra
  # targets: prometheus | grafana | altermanager | loki | nginx
  # labels: [ip, instance, type]
  # path: targets/infra/<ip>.yml
  #--------------------------------------------------------------#
  - job_name: infra
    metrics_path: /metrics
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/infra/*.yml]

  #--------------------------------------------------------------#
  # job: node
  # node_exporter, haproxy, docker, promtail
  # labels: [cls, ins, ip, instance]
  # path: targets/pgsql/<ip>.yml
  #--------------------------------------------------------------#
  - job_name: node
    metrics_path: /metrics
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/node/*.yml]


  #--------------------------------------------------------------#
  # job: mysql
  # labels: [cls, ins, ip, instance]
  # path: targets/mysql/<mysql_instance>.yml
  #--------------------------------------------------------------#
  - job_name: mysql
    metrics_path: /metrics
    file_sd_configs:
      - refresh_interval: 5s
        files: [/etc/prometheus/targets/mysql/*.yml]


...
EOF
```

```bash
chown prometheus:prometheus /etc/prometheus/prometheus.yml && \
chmod 0644 /etc/prometheus/prometheus.yml
```

### Prometheus bin scripts

```bash
cat <<EOF | tee /etc/prometheus/bin/check
#!/bin/bash
promtool check config /etc/prometheus/prometheus.yml
EOF

cat <<EOF | tee /etc/prometheus/bin/new
#!/bin/bash

echo "destroy prometheus data and create a new one"
systemctl stop prometheus
rm -rf /data/prometheus/data/*
systemctl start prometheus

echo "prometheus recreated"
systemctl status prometheus
EOF

cat <<EOF | tee /etc/prometheus/bin/reload
#!/bin/bash
ps aux | grep prometheus | grep 'config.file' | awk '{print \$2}' | xargs -n1 kill -s SIGHUP
EOF

cat <<EOF | tee /etc/prometheus/bin/status
#!/bin/bash

systemctl status prometheus
ps aux | grep prometheus
EOF

chown -R prometheus:prometheus /etc/prometheus/bin && \
chmod -R 0755 /etc/prometheus/bin
```

### Prometheus rules

copy prometheus rules to `/etc/prometheus/rules`

```yaml title="/etc/prometheus/rules/infra.yml"
cat <<EOF | tee /etc/prometheus/rules/infra.yml
groups:
  - name: infra_rules
    rules: 
      - record: infra_up
        expr: up{job="infra"}

      - record: agent_up
        expr: up{job!~"infra|node|etcd|minio|pgsql|redis|mongo|mysql"}

  - name: infra_alert
    rules:
      - alert: InfraDown
        expr: infra_up < 1
        for: 1m
        labels: { level: 0, serverity: CRIT, category: infra }
        annotations:
          summary: "CRIT InfraDown {{ \$labels.type }}@{{ \$labels.instance }}"
          description: |
            infra_up[type={{ \$labels.type }}, instance={{ \$labels.instance }}] = {{ \$value | printf "%.2f" }} < 1

      - alert: AgentDown
        expr: agent_up < 1
        for: 1m
        labels: { level: 0, severity: CRIT, category: infra }
        annotations:
          summary: "CRIT AgentDown {{ \$labels.ins }}@{{ \$labels.instance }}"
          description: |
            agent_up[ins={{ \$labels.ins }}, instance={{ \$labels.instance }}] = {{ \$value | printf "%.2f" }} < 1
EOF
```

```yaml title="/etc/prometheus/rules/node.yml"
cat <<EOF | tee /etc/prometheus/rules/node.yml
groups:

  ################################################################
  #                   Node Derived Metrics                       #
  ################################################################
  # node & load balancer (haproxy) are treated as infrastructure
  # monitoring components: prometheus, alertmanager, grafana

  - name: node_rules
    rules:

      ################################################################
      #                          Aliveness                           #
      ################################################################


      #==============================================================#
      #                      Node & LB Uptime                        #
      #==============================================================#
      # seconds since node bootstrap
      - record: node:ins:uptime
        expr: time() - node_boot_time_seconds


      ################################################################
      #                         Membership                           #
      ################################################################
      # node:ins holds identity mapping information ID(hostname,ins,ip) -> INS (instance identifier)
      - record: node:ins
        expr: |
          sum without (machine, release, version, sysname, domainname) (
            label_replace(node_uname_info, "id", "\$1", "nodename", "(.+)") OR
            label_replace(node_uname_info, "id", "\$1", "instance", "(.+)\\\:\\\d+") OR
            label_replace(node_uname_info, "id", "\$1", "ins", "(.+)")
          )

      ################################################################
      #                  Node : CPU & Schedule                       #
      ################################################################

      #--------------------------------#
      #           CPU Core             #
      #--------------------------------#
      # metrics about single cpu core

      # time spent per second for single cpu core on specific mode
      # {cpu,mode}
      - record: node:cpu:time_irate1m
        expr: irate(node_cpu_seconds_total[1m])

      # {cpu} total time spent per second on single cpu 
      - record: node:cpu:total_time_irate1m
        expr: sum without (mode) (node:cpu:time_irate1m)

      # {cpu} idle time spent per second on single cpu
      - record: node:cpu:idle_time_irate1m
        expr: sum without (mode) (node:cpu:time_irate1m{mode="idle"})

      # {cpu} realtime cpu usage per cpu, core metric
      - record: node:cpu:usage
        expr: 1 - node:cpu:idle_time_irate1m / node:cpu:total_time_irate1m

      # average 1m,5m,15m usage of single cpu core
      - record: node:cpu:usage_avg1m
        expr: avg_over_time(node:cpu:usage[1m])
      - record: node:cpu:usage_avg5m
        expr: avg_over_time(node:cpu:usage[5m])
      - record: node:cpu:usage_avg15m
        expr: avg_over_time(node:cpu:usage[15m])

      #--------------------------------#
      #           CPU Count            #
      #--------------------------------#
      # number of cpu cores of this node
      - record: node:ins:cpu_count
        expr: count without (cpu) (node:cpu:usage)
      - record: node:cls:cpu_count
        expr: sum by (cls, job) (node:ins:cpu_count)
      - record: node:env:cpu_count
        expr: sum by (job) (node:ins:cpu_count)

      #--------------------------------#
      #      CPU Usage (Realtime)      #
      #--------------------------------#
      - record: node:ins:cpu_usage
        expr: avg without (cpu) (node:cpu:usage)
      - record: node:cls:cpu_usage
        expr: avg by (cls, job) (node:cpu:usage)
      - record: node:env:cpu_usage
        expr: sum by (job) (node:cls:cpu_usage * node:cls:cpu_count) / sum by (job) (node:cls:cpu_count)

      #--------------------------------#
      # CPU Usage (realtime|1m|5m|15m) #
      #--------------------------------#
      # cpu usage of single node|instance
      - record: node:ins:cpu_usage_1m
        expr: avg_over_time(node:ins:cpu_usage[1m])
      - record: node:ins:cpu_usage_5m
        expr: avg_over_time(node:ins:cpu_usage[5m])
      - record: node:ins:cpu_usage_15m
        expr: avg_over_time(node:ins:cpu_usage[15m])

      # cpu usage of single cluster
      - record: node:cls:cpu_usage_1m
        expr: avg_over_time(node:cls:cpu_usage[1m])
      - record: node:cls:cpu_usage_5m
        expr: avg_over_time(node:cls:cpu_usage[5m])
      - record: node:cls:cpu_usage_15m
        expr: avg_over_time(node:cls:cpu_usage[15m])

      # cpu usage of single environment (identified by job)
      - record: node:env:cpu_usage_1m
        expr: avg_over_time(node:env:cpu_usage[1m])
      - record: node:env:cpu_usage_5m
        expr: avg_over_time(node:env:cpu_usage[5m])
      - record: node:env:cpu_usage_15m
        expr: avg_over_time(node:env:cpu_usage[15m])

      #--------------------------------#
      #            Schedule            #
      #--------------------------------#
      # cpu schedule time-slices by (cpu,ins)
      - record: node:cpu:sched_timeslices_rate1m
        expr: rate(node_schedstat_timeslices_total[1m])
      - record: node:ins:sched_timeslices_rate1m
        expr: sum without (cpu) (node:cpu:sched_timeslices)

      # cpu average waiting on schedule
      - record: node:cpu:sched_wait_rate1m
        expr: rate(node_schedstat_waiting_seconds_total[1m])
      - record: node:ins:sched_wait_rate1m
        expr: avg without (cpu) (node:cpu:sched_wait_rate1m)

      # process fork rate1m
      - record: node:ins:forks_rate1m
        expr: rate(node_forks_total[1m])

      # interrupt rate
      - record: node:ins:interrupt_rate1m
        expr: rate(node_intr_total[1m])

      # context switch rate
      - record: node:ins:ctx_switch_rate1m
        expr: rate(node_context_switches_total{}[1m])


      #--------------------------------#
      #             Load               #
      #--------------------------------#
      # normalized load 1,5,15 (load divide by cpu)
      - record: node:ins:stdload1
        expr: node_load1 / node:ins:cpu_count
      - record: node:ins:stdload5
        expr: node_load5 / node:ins:cpu_count
      - record: node:ins:stdload15
        expr: node_load15 / node:ins:cpu_count

      - record: node:cls:stdload1
        expr: sum by (cls, job) (node_load1) / node:cls:cpu_count
      - record: node:cls:stdload5
        expr: sum by (cls, job) (node_load5) / node:cls:cpu_count
      - record: node:cls:stdload15
        expr: sum by (cls, job) (node_load15) / node:cls:cpu_count

      - record: node:env:stdload1
        expr: sum by (job) (node:cls:stdload1) / node:env:cpu_count
      - record: node:env:stdload5
        expr: sum by (job) (node:cls:stdload5) / node:env:cpu_count
      - record: node:env:stdload15
        expr: sum by (job) (node:cls:stdload15) / node:env:cpu_count


      ################################################################
      #                   Node : Memory & Swap                       #
      ################################################################

      #--------------------------------#
      #       Physical Memory          #
      #--------------------------------#
      # kernel memory size
      - record: node:ins:mem_kernel
        expr: |
          node_memory_KernelStack_bytes +
          node_memory_VmallocUsed_bytes +
          node_memory_HardwareCorrupted_bytes +
          node_memory_PageTables_bytes +
          node_memory_Slab_bytes +
          node_memory_PageTables_bytes

      # application rss memory size
      - record: node:ins:mem_rss
        expr: |
          node_memory_MemTotal_bytes -
          (node_memory_KernelStack_bytes +
          node_memory_VmallocUsed_bytes +
          node_memory_HardwareCorrupted_bytes +
          node_memory_PageTables_bytes +
          node_memory_Slab_bytes +
          node_memory_PageTables_bytes) - 
          node_memory_Buffers_bytes -
          (node_memory_Hugepagesize_bytes * node_memory_HugePages_Total) -
          (node_memory_Cached_bytes - node_memory_Mapped_bytes ) -
          node_memory_MemFree_bytes

      # mem avail: avail / total
      - record: node:ins:mem_avail
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

      # mem usage: 1 - avail / total
      - record: node:ins:mem_usage
        expr: 1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

      # swap usage by instance (NaN if swap is disabled)
      - record: node:ins:swap_usage
        expr: 1 - node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes

      # instance commit ratio: commit_as / commit_limit
      - record: node:ins:mem_commit_ratio
        expr: node_memory_Committed_AS_bytes / node_memory_CommitLimit_bytes

      # cluster memory usage agg
      - record: node:cls:mem_usage
        expr: 1 - sum by (cls, job) (node_memory_MemAvailable_bytes) / sum by (cls, job) (node_memory_MemTotal_bytes)

      # environment stats
      - record: node:env:mem_total
        expr: sum by (job) (node_memory_MemTotal_bytes)
      - record: node:env:mem_avail
        expr: sum by (job) (node_memory_MemAvailable_bytes)
      - record: node:env:mem_usage
        expr: 1 - node:env:mem_avail / node:env:mem_total

      #--------------------------------#
      #        Virtual  Memory         #
      #--------------------------------#
      # page fault (mem page missing)
      - record: node:ins:pagefault_rate1m
        expr: rate(node_vmstat_pgfault[1m])

      # major page fault
      - record: node:ins:pgmajfault_rate1m
        expr: rate(node_vmstat_pgmajfault[1m])

      # page in (disk to mem)
      - record: node:ins:pagein_rate1m
        expr: rate(node_vmstat_pgpgin[1m])

      # page out (mem to disk)
      - record: node:ins:pageout_rate1m
        expr: rate(node_vmstat_pgpgout[1m])

      # page swap in (swap disk to mem)
      - record: node:ins:swapin_rate1m
        expr: rate(node_vmstat_pswpin[1m])

      # page swap out (swap mem to disk)
      - record: node:ins:swapout_rate1m
        expr: rate(node_vmstat_pswpout[1m])



      ################################################################
      #                 Node : Disk & Filesystem                     #
      ################################################################
      # rootfs & tmpfs are excluded in metrics calculation

      #--------------------------------#
      #            Disk Util           #
      #--------------------------------#
      - record: node:dev:disk_util_1m
        expr: rate(node_disk_io_time_seconds_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])

      - record: node:dev:disk_avg_queue_size
        expr: rate(node_disk_io_time_weighted_seconds_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])


      #--------------------------------#
      #        Disk Read/Writes        #
      #--------------------------------#

      # disk reads request per second in last 1 minute
      - record: node:dev:disk_reads_rate1m
        expr: rate(node_disk_reads_completed_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:ins:disk_reads_rate1m
        expr: sum without (device) (node:dev:disk_reads_rate1m)
      - record: node:cls:disk_reads_rate1m
        expr: sum by (cls, job) (node:ins:disk_reads_rate1m)

      # disk write request per second in last 1 minute
      - record: node:dev:disk_writes_rate1m
        expr: rate(node_disk_writes_completed_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:ins:disk_writes_rate1m
        expr: sum without (device) (node:dev:disk_writes_rate1m)
      - record: node:cls:disk_writes_rate1m
        expr: sum by (cls, job) (node:ins:disk_writes_rate1m)

      # disk merged reads request per second in last 1 minute
      - record: node:dev:disk_mreads_rate1m
        expr: rate(node_disk_reads_merged_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:ins:disk_mreads_rate1m
        expr: sum without (device) (node:dev:disk_mreads_rate1m)
      - record: node:cls:disk_mreads_rate1m
        expr: sum by (cls, job) (node:ins:disk_mreads_rate1m)

      # disk merged write request per second in last 1 minute
      - record: node:dev:disk_mwrites_rate1m
        expr: rate(node_disk_writes_merged_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:ins:disk_mwrites_rate1m
        expr: sum without (device) (node:dev:disk_mwrites_rate1m)
      - record: node:cls:disk_mwrites_rate1m
        expr: sum by (cls, job) (node:ins:disk_mwrites_rate1m)

      # disk i/o request per second in last 1 minute
      - record: node:dev:disk_iops_1m
        expr: node:dev:disk_reads_rate1m + node:dev:disk_writes_rate1m
      - record: node:ins:disk_iops_1m
        expr: node:ins:disk_reads_rate1m + node:ins:disk_writes_rate1m
      - record: node:cls:disk_iops_1m
        expr: node:cls:disk_reads_rate1m + node:cls:disk_writes_rate1m

      # merged read ratio
      - record: node:dev:disk_mreads_ratio1m
        expr: node:dev:disk_mreads_rate1m / (node:dev:disk_reads_rate1m + node:dev:disk_mreads_rate1m)
      - record: node:ins:disk_mreads_ratio1m
        expr: node:ins:disk_mreads_rate1m / (node:ins:disk_reads_rate1m + node:ins:disk_mreads_rate1m)
      - record: node:cls:disk_mreads_ratio1m
        expr: node:cls:disk_mreads_rate1m / (node:cls:disk_reads_rate1m + node:cls:disk_mreads_rate1m)

      # merged write ratio
      - record: node:dev:disk_mwrites_ratio1m
        expr: node:dev:disk_mwrites_rate1m / (node:dev:disk_writes_rate1m + node:dev:disk_mwrites_rate1m)
      - record: node:ins:disk_mwrites_ratio1m
        expr: node:ins:disk_mwrites_rate1m / (node:ins:disk_writes_rate1m + node:ins:disk_mwrites_rate1m)
      - record: node:cls:disk_mwrites_ratio1m
        expr: node:cls:disk_mwrites_rate1m / (node:cls:disk_writes_rate1m + node:cls:disk_mwrites_rate1m)


      #--------------------------------#
      #           Disk Bytes           #
      #--------------------------------#
      # read bandwidth (rate1m)
      - record: node:dev:disk_read_bytes_rate1m
        expr: rate(node_disk_read_bytes_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:ins:disk_read_bytes_rate1m
        expr: sum without (device) (node:dev:disk_read_bytes_rate1m)
      - record: node:cls:disk_read_bytes_rate1m
        expr: sum by (cls, job) (node:ins:disk_read_bytes_rate1m)

      # write bandwidth (rate1m)
      - record: node:dev:disk_write_bytes_rate1m
        expr: rate(node_disk_written_bytes_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:ins:disk_write_bytes_rate1m
        expr: sum without (device) (node:dev:disk_write_bytes_rate1m)
      - record: node:cls:disk_write_bytes_rate1m
        expr: sum by (cls, job) (node:ins:disk_write_bytes_rate1m)

      # io bandwidth (rate1m)
      - record: node:dev:disk_io_bytes_rate1m
        expr: node:dev:disk_read_bytes_rate1m + node:dev:disk_write_bytes_rate1m
      - record: node:ins:disk_io_bytes_rate1m
        expr: node:ins:disk_read_bytes_rate1m + node:ins:disk_write_bytes_rate1m
      - record: node:cls:disk_io_bytes_rate1m
        expr: node:cls:disk_read_bytes_rate1m + node:cls:disk_write_bytes_rate1m

      #--------------------------------#
      #           Disk Time            #
      #--------------------------------#
      - record: node:dev:disk_read_time_rate1m
        expr: rate(node_disk_read_time_seconds_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:dev:disk_write_time_rate1m
        expr: rate(node_disk_write_time_seconds_total{device=~"sd.*|vd.*|hd.*|nvme.*"}[1m])
      - record: node:dev:disk_io_time_rate1m
        expr: node:dev:disk_read_time_rate1m + node:dev:disk_write_time_rate1m

      #--------------------------------#
      #           Disk RT              #
      #--------------------------------#
      - record: node:dev:disk_read_rt_1m
        expr: node:dev:disk_read_time_rate1m / node:dev:disk_reads_rate1m
      - record: node:dev:disk_write_rt_1m
        expr: node:dev:disk_write_time_rate1m / node:dev:disk_writes_rate1m
      - record: node:dev:disk_io_rt_1m
        expr: node:dev:disk_io_time_rate1m / node:dev:disk_iops_1m

      #--------------------------------#
      #        Disk I/O Batch          #
      #--------------------------------#
      - record: node:dev:disk_read_batch_1m
        expr: node:dev:disk_read_bytes_rate1m / node:dev:disk_reads_rate1m
      - record: node:dev:disk_write_batch_1m
        expr: node:dev:disk_write_bytes_rate1m / node:dev:disk_writes_rate1m
      - record: node:dev:disk_io_batch_1m
        expr: node:dev:disk_io_bytes_rate1m / node:dev:disk_iops_1m

      #--------------------------------#
      #           Filesystem           #
      #--------------------------------#
      # filesystem space metrics
      - record: node:fs:free_bytes
        expr: node_filesystem_free_bytes{fstype!~"(n|root|tmp)fs.*"}
      - record: node:fs:avail_bytes
        expr: node_filesystem_avail_bytes{fstype!~"(n|root|tmp)fs.*"}
      - record: node:fs:size_bytes
        expr: node_filesystem_size_bytes{fstype!~"(n|root|tmp)fs.*"}

      # instance level
      - record: node:ins:free_bytes
        expr: sum without(device,mountpoint,fstype) (node:fs:free_bytes)
      - record: node:ins:avail_bytes
        expr: sum without(device,mountpoint,fstype) (node:fs:avail_bytes)
      - record: node:ins:size_bytes
        expr: sum without(device,mountpoint,fstype) (node:fs:size_bytes)

      # cluster level
      - record: node:cls:free_bytes
        expr: sum by (job,cls) (node:ins:free_bytes)
      - record: node:cls:avail_bytes
        expr: sum by (job,cls) (node:ins:avail_bytes)
      - record: node:cls:size_bytes
        expr: sum by (job,cls) (node:ins:size_bytes)

      # environment level
      - record: node:env:free_bytes
        expr: sum by(job) (node:ins:free_bytes)
      - record: node:env:avail_bytes
        expr: sum by(job) (node:ins:avail_bytes)
      - record: node:env:size_bytes
        expr: sum by(job) (node:ins:size_bytes)

      # filesystem space usage ( 1 - avail/total )
      - record: node:fs:space_usage
        expr: 1 - node:fs:avail_bytes / node:fs:size_bytes
      - record: node:cls:space_usage
        expr: 1 - node:cls:avail_bytes / node:cls:size_bytes
      - record: node:env:space_usage
        expr: 1 - node:env:avail_bytes / node:env:size_bytes

      # max space usage
      - record: node:ins:space_usage_max
        expr: max without (device, fstype, mountpoint) (node:fs:space_usage)
      - record: node:cls:space_usage_max
        expr: max by (job, cls) (node:ins:space_usage_max)

      # max space usage by device
      - record: node:env:device_space_usage_max
        expr: max by (job, device, fstype, mountpoint) (node:fs:space_usage)

      # space delta and prediction
      - record: node:fs:space_deriv1h
        expr: deriv(node:fs:avail_bytes[1h])

      # estimated space exhaust time according to last 1h space deriv, clamp into (-1, 126144000)
      # -1      : free space are increased in last 1h (therefore will NOT exhaust)
      # (-1,0)  : not likely happen, just ignore
      # (0,max) : indicate estimated seconds running out of space
      # max     : at most 4 years (1460 days) (avoid useless Inf)
      - record: node:fs:space_exhaust
        expr: clamp(node:fs:avail_bytes / - node:fs:space_deriv1h, -1, 126230400)

      # predict free-space 1d later according to last 1h's activity
      - record: node:fs:space_predict_1d
        expr: predict_linear(node:fs:avail_bytes[1h], 86400)


      #--------------------------------#
      #              iNode             #
      #--------------------------------#
      # free inodes of this filesystem
      - record: node:fs:inode_free
        expr: node_filesystem_files_free{fstype!~"(n|root|tmp)fs.*"}

      # total inodes of this filesystem
      - record: node:fs:inode_total
        expr: node_filesystem_files{fstype!~"(n|root|tmp)fs.*"}

      # used inodes of this filesystem
      - record: node:fs:inode_used
        expr: node:fs:inode_total - node:fs:inode_free

      # inode usage of this filesystem
      - record: node:fs:inode_usage
        expr: 1 - (node:fs:inode_free / node:fs:inode_total)

      # overall inode usage (usually max(node:fs:inode_usage) would be a better agg)
      - record: node:ins:inode_usage
        expr: |
          sum without (fstype, device, mountpoint) (node:fs:inode_used) /
          sum without (fstype, device, mountpoint) (node:fs:inode_total)

      #--------------------------------#
      #         File Descriptor        #
      #--------------------------------#
      # file descriptor usage
      - record: node:ins:fd_usage
        expr: node_filefd_allocated / node_filefd_maximum

      - record: node:ins:fd_alloc_rate1m
        expr: rate(node_filefd_allocated[1m])



      ################################################################
      #                 Node : Network & Protocol                    #
      ################################################################

      #--------------------------------#
      #       Network Interface        #
      #--------------------------------#

      # transmit pps
      - record: node:dev:network_tx_pps1m
        expr: rate(node_network_transmit_packets_total[1m])
      - record: node:ins:network_tx_pps1m
        expr: sum without (device) (node:dev:network_tx_pps1m{device!~"lo|bond.*"})
      - record: node:cls:network_tx_pps1m
        expr: sum by (cls, job) (node:ins:network_tx_pps1m)

      # receive pps
      - record: node:dev:network_rx_pps1m
        expr: rate(node_network_receive_packets_total[1m])
      - record: node:ins:network_rx_pps1m
        expr: sum without (device) (node:dev:network_rx_pps1m{device!~"lo|bond.*"})
      - record: node:cls:network_rx_pps1m
        expr: sum by (cls, job) (node:ins:network_rx_pps1m)

      # transmit bandwidth (out)
      - record: node:dev:network_tx_bytes_rate1m
        expr: rate(node_network_transmit_bytes_total[1m])
      - record: node:ins:network_tx_bytes_rate1m
        expr: sum without (device) (node:dev:network_tx_bytes_rate1m{device!~"lo|bond.*"})
      - record: node:cls:network_tx_bytes_rate1m
        expr: sum by (cls, job) (node:ins:network_tx_bytes_rate1m)

      # receive bandwidth (in)
      - record: node:dev:network_rx_bytes_rate1m
        expr: rate(node_network_receive_bytes_total[1m])
      - record: node:ins:network_rx_bytes_rate1m
        expr: sum without (device) (node:dev:network_rx_bytes_rate1m{device!~"lo|bond.*"})
      - record: node:cls:network_rx_bytes_rate1m
        expr: sum by (cls, job) (node:ins:network_rx_bytes_rate1m)

      # io(tx+rx) bandwidth
      - record: node:dev:network_io_bytes_rate1m
        expr: node:dev:network_tx_bytes_rate1m + node:dev:network_rx_bytes_rate1m
      - record: node:ins:network_io_bytes_rate1m
        expr: node:ins:network_tx_bytes_rate1m + node:ins:network_rx_bytes_rate1m
      - record: node:cls:network_io_bytes_rate1m
        expr: node:cls:network_tx_bytes_rate1m + node:cls:network_rx_bytes_rate1m

      #--------------------------------#
      #        TCP/IP Protocol         #
      #--------------------------------#
      # tcp segments in (rate1m)
      - record: node:ins:tcp_insegs_rate1m
        expr: rate(node_netstat_Tcp_InSegs[1m])

      # tcp segments out (rate1m)
      - record: node:ins:tcp_outsegs_rate1m
        expr: rate(node_netstat_Tcp_OutSegs[1m])

      # tcp segments retransmit (rate1m)
      - record: node:ins:tcp_retranssegs_rate1m
        expr: rate(node_netstat_Tcp_RetransSegs[1m])

      # tcp segments (i/o) (rate1m)
      - record: node:ins:tcp_segs_rate1m
        expr: node:ins:tcp_insegs_rate1m + node:ins:tcp_outsegs_rate1m

      # tcp retransmit rate (last 1m)
      - record: node:ins:tcp_retrans_ratio1m
        expr: node:ins:tcp_retranssegs_rate1m / node:ins:tcp_outsegs_rate1m

      # tcp error count
      - record: node:ins:tcp_error
        expr: |
            node_netstat_TcpExt_ListenOverflows +
            node_netstat_TcpExt_ListenDrops +
            node_netstat_Tcp_InErrs

      # tcp error (rate1m)
      - record: node:ins:tcp_error_rate1m
        expr: rate(node:ins:tcp_error[1m])

      # tcp passive open (rate1m)
      - record: node:ins:tcp_passive_opens_rate1m
        expr: rate(node_netstat_Tcp_PassiveOpens[1m])

      # tcp active open (rate1m)
      - record: node:ins:tcp_active_opens_rate1m
        expr: rate(node_netstat_Tcp_ActiveOpens[1m])

      # tcp close (rate1m)
      - record: node:ins:tcp_attempt_fails_rate1m
        expr: rate(node_netstat_Tcp_AttemptFails[1m])

      # tcp establish (rate1m)
      - record: node:ins:tcp_estab_resets_rate1m
        expr: rate(node_netstat_Tcp_EstabResets[1m])

      # tcp overflow (rate1m)
      - record: node:ins:tcp_overflow_rate1m
        expr: rate(node_netstat_TcpExt_ListenOverflows[1m])

      # tcp dropped (rate1m)
      - record: node:ins:tcp_dropped_rate1m
        expr: rate(node_netstat_TcpExt_ListenDrops[1m])

      # udp segments in (rate1m)
      - record: node:ins:udp_in_rate1m
        expr: rate(node_netstat_Udp_InDatagrams{}[1m])

      # udp segments out (rate1m)
      - record: node:ins:udp_out_rate1m
        expr: rate(node_netstat_Udp_OutDatagrams[1m])

      ################################################################
      #                    Node : Miscellaneous                      #
      ################################################################

      #--------------------------------#
      #              Time              #
      #--------------------------------#
      - record: node:ins:time_drift
        expr: abs(node_timex_offset_seconds)

      - record: node:cls:time_drift_max
        expr: max by (cls, job) (node:ins:time_drift)

      - record: node:cls:time_drift_range
        expr: max by (cls, job) (node_ntp_offset_seconds) - min by (cls, job) (node_ntp_offset_seconds)


      ################################################################
      #                          HAProxy                             #
      ################################################################
      # cpu usage (busy ratio) of haproxy instance
      - record: haproxy:ins:usage
        expr: (100 - haproxy_process_idle_time_percent) / 100

      - record: haproxy:cls:usage
        expr: avg by (cls, job) (haproxy:ins:usage)

      - record: haproxy:ins:uptime
        expr: time() - haproxy_process_start_time_seconds

      # SLI: aliveness
      - record: haproxy:ins:service_up
        expr: sum by (ins,proxy) (haproxy_server_status{state="UP", proxy!='stats'}) > bool 0
      - record: haproxy:cls:service_up
        expr: sum by (cls,proxy) (haproxy_server_status{state="UP", proxy!='stats'}) > bool 0




      ################################################################
      #                          Docker                              #
      ################################################################

      ################################################################
      #                          Promtail                            #
      ################################################################



  ################################################################
  #                         Node Alert                           #
  ################################################################
  - name: node_alert
    rules:

      #==============================================================#
      #                          Aliveness                           #
      #==============================================================#
      # node exporter is dead indicate node is down
      - alert: NodeDown
        expr: node_up < 1
        for: 1m
        labels: { level: 0, severity: CRIT, category: node }
        annotations:
          summary: "CRIT NodeDown {{ \$labels.ins }}@{{ \$labels.instance }}"
          description: |
            node_up[ins={{ \$labels.ins }}, instance={{ \$labels.instance }}] = {{ \$value }} < 1
            http://g.kubegg/d/node-instance?var-ins={{ \$labels.ins }}

      # haproxy the load balancer
      - alert: HaproxyDown
        expr: haproxy_up < 1
        for: 1m
        labels: { level: 0, severity: CRIT, category: node }
        annotations:
          summary: "CRIT HaproxyDown {{ \$labels.ins }}@{{ \$labels.instance }}"
          description: |
            haproxy_up[ins={{ \$labels.ins }}, instance={{ \$labels.instance }}] = {{ \$value }} < 1
            http://g.kubegg/d/node-haproxy?var-ins={{ \$labels.ins }}

      # promtail the logging agent
      - alert: PromtailDown
        expr: promtail_up < 1
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: "CRIT PromtailDown {{ \$labels.ins }}@{{ \$labels.instance }}"
          description: |
            promtail_up[ins={{ \$labels.ins }}, instance={{ \$labels.instance }}] = {{ \$value }} < 1
            http://g.kubegg/d/node-instance?var-ins={{ \$labels.ins }}

      # docker the container engine
      - alert: DockerDown
        expr: docker_up < 1
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: "CRIT DockerDown {{ \$labels.ins }}@{{ \$labels.instance }}"
          description: |
            docker_up[ins={{ \$labels.ins }}, instance={{ \$labels.instance }}] = {{ \$value }} < 1
            http://g.kubegg/d/node-instance?var-ins={{ \$labels.ins }}

      # keepalived daemon
      - alert: KeepalivedDown
        expr: keepalived_up < 1
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: "CRIT KeepalivedDown {{ \$labels.ins }}@{{ \$labels.instance }}"
          description: |
            keepalived_up[ins={{ \$labels.ins }}, instance={{ \$labels.instance }}] = {{ \$value }} < 1
            http://g.kubegg/d/node-instance?var-ins={{ \$labels.ins }}

  

      #==============================================================#
      #                          Node : CPU                          #
      #==============================================================#
      # cpu usage high : 1m avg cpu usage > 70% for 3m
      - alert: NodeCpuHigh
        expr: node:ins:cpu_usage_1m > 0.70
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeCpuHigh {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            node:ins:cpu_usage[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.2f" }} > 70%

      # OPTIONAL: one core high
      # OPTIONAL: throttled
      # OPTIONAL: frequency
      # OPTIONAL: steal

      #==============================================================#
      #                       Node : Schedule                        #
      #==============================================================#
      # node load high : 1m avg standard load > 100% for 3m
      - alert: NodeLoadHigh
        expr: node:ins:stdload1 > 1
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeLoadHigh {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            node:ins:stdload1[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.2f" }} > 100%


      #==============================================================#
      #                        Node : Memory                         #
      #==============================================================#
      # available memory < 10%
      - alert: NodeOutOfMem
        expr: node:ins:mem_avail < 0.10
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeOutOfMem {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            node:ins:mem_avail[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.2f" }} < 10%

      # commit ratio > 90%
      #- alert: NodeMemCommitRatioHigh
      #  expr: node:ins:mem_commit_ratio > 0.90
      #  for: 1m
      #  labels: { level: 1, severity: WARN, category: node }
      #  annotations:
      #    summary: 'WARN NodeMemCommitRatioHigh {{ $labels.ins }}@{{ $labels.instance }} {{ $value  | printf "%.2f" }}'
      #    description: |
      #      node:ins:mem_commit_ratio[ins={{ $labels.ins }}] = {{ $value  | printf "%.2f" }} > 90%

      # OPTIONAL: EDAC Errors

      #==============================================================#
      #                        Node : Swap                           #
      #==============================================================#
      # swap usage > 1%
      - alert: NodeMemSwapped
        expr: node:ins:swap_usage > 0.01
        for: 5m
        labels: { level: 2, severity: INFO, category: node }
        annotations:
          summary: 'INFO NodeMemSwapped {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            node:ins:swap_usage[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.2f" }} > 1%

      #==============================================================#
      #                     Node : File System                       #
      #==============================================================#

      # filesystem usage > 90%
      - alert: NodeFsSpaceFull
        expr: node:fs:space_usage > 0.90
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeFsSpaceFull {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            node:fs:space_usage[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.2f" }} > 90%

      # inode usage > 90%
      - alert: NodeFsFilesFull
        expr: node:fs:inode_usage > 0.90
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeFsFilesFull {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            node:fs:inode_usage[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.2f" }} > 90%

      # file descriptor usage > 90%
      - alert: NodeFdFull
        expr: node:ins:fd_usage > 0.90
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeFdFull {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            node:ins:fd_usage[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.2f" }} > 90%

      # OPTIONAL: space predict 1d
      # OPTIONAL: filesystem read-only
      # OPTIONAL: fast release on disk space

      #==============================================================#
      #                          Node : Disk                         #
      #==============================================================#
      # read latency > 32ms (typical on pci-e ssd: 100µs)
      - alert: NodeDiskSlow
        expr: node:dev:disk_read_rt_1m{device="dfa"} > 0.032 or node:dev:disk_write_rt_1m{device="dfa"} > 0.032
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeReadSlow {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.6f" }}'
          description: |
            node:dev:disk_read_rt_1m[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.6f" }} > 32ms

      # OPTIONAL: raid card failure
      # OPTIONAL: read/write traffic high
      # OPTIONAL: read/write latency high

      #==============================================================#
      #                        Node : Network                        #
      #==============================================================#
      # OPTIONAL: unusual network traffic
      # OPTIONAL: interface saturation high

      #==============================================================#
      #                        Node : Protocol                       #
      #==============================================================#

      # rate(node:ins:tcp_error[1m]) > 1
      - alert: NodeTcpErrHigh
        expr: rate(node:ins:tcp_error[1m]) > 1
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeTcpErrHigh {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.2f" }}'
          description: |
            rate(node:ins:tcp_error{ins={{ \$labels.ins }}}[1m]) = {{ \$value  | printf "%.2f" }} > 1

      # node:ins:tcp_retrans_ratio1m > 1e-4
      - alert: NodeTcpRetransHigh
        expr: node:ins:tcp_retrans_ratio1m > 1e-2
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'INFO NodeTcpRetransHigh {{ \$labels.ins }}@{{ \$labels.instance }} {{ \$value  | printf "%.6f" }}'
          description: |
            node:ins:tcp_retrans_ratio1m[ins={{ \$labels.ins }}] = {{ \$value  | printf "%.6f" }} > 1%

      # OPTIONAL: tcp conn high
      # OPTIONAL: udp traffic high
      # OPTIONAL: conn track

      #==============================================================#
      #                          Node : Time                         #
      #==============================================================#

      - alert: NodeTimeDrift
        expr: node_timex_sync_status != 1
        for: 1m
        labels: { level: 1, severity: WARN, category: node }
        annotations:
          summary: 'WARN NodeTimeDrift {{ \$labels.ins }}@{{ \$labels.instance }}'
          description: |
            node_timex_status[ins={{ \$labels.ins }}]) = {{ \$value | printf "%.6f" }} != 0 or
            node_timex_sync_status[ins={{ \$labels.ins }}]) = {{ \$value | printf "%.6f" }} != 1


      # time drift > 64ms
      # - alert: NodeTimeDrift
      #   expr: node:ins:time_drift > 0.064
      #   for: 1m
      #   labels: { level: 1, severity: WARN, category: node }
      #   annotations:
      #     summary: 'WARN NodeTimeDrift {{ $labels.ins }}@{{ $labels.instance }}'
      #     description: |
      #       abs(node_timex_offset_seconds)[ins={{ $labels.ins }}]) = {{ $value | printf "%.6f" }} > 64ms

...
EOF
```

```bash
chown -R prometheus:prometheus /etc/prometheus/rules && \
chmod -R 0644 /etc/prometheus/rules
```

### Prometheus 启动

```bash
systemctl enable prometheus
systemctl restart prometheus

systemctl daemon-reload
systemctl reload prometheus
```

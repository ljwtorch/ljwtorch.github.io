> [!TIP]
> 下方采集指标内容只写正在使用的一些指标名称，其他部分自行搭建后进行了解！

## 1.node-exporter

### 1.1 部署方式

**单机部署**

```shell
docker run -d --name node-exporter --restart=always --net="host" --pid="host" -v "/:/host:ro,rslave" quay.io/prometheus/node-exporter:latest --path.rootfs=/host
```

**docker compose 部署**

```yaml
  node-exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node-exporter
    restart: always
    network_mode: host
    pid: host
    volumes:
      - /:/host:ro,rslave
    command: --path.rootfs=/host
    depends_on:
      - prometheus
```

访问 ip:9100

### 1.2 采集指标

1. CPU使用率：`round((1 - avg(rate(node_cpu_seconds_total{instance="instance:9100",mode="idle"}[1m])) by (instance)) * 100, 0.01)`
2. RAM使用率：`round((1 - (node_memory_MemAvailable_bytes{instance="instance:9100"} / (node_memory_MemTotal_bytes{instance="instance:9100"})))* 100,0.01)`
3. 磁盘使用率：`round((node_filesystem_size_bytes{instance="instance:9100",mountpoint="/"}-node_filesystem_free_bytes{instance="instance:9100",mountpoint="/"})*100/(node_filesystem_avail_bytes{instance="instance:9100",mountpoint="/"}+(node_filesystem_size_bytes{instance="instance:9100",mountpoint="/"}-node_filesystem_free_bytes{instance="instance:9100",mountpoint="/"})),0.01)`
4. 网络连接数使用量：`node_netstat_Tcp_CurrEstab{instance="instance:9100"}`



## 2.blackbox-exporter

### 2.1 部署方式

**docker compose部署**

```yaml
  blackbox-exporter:
    image: quay.io/prometheus/blackbox-exporter:latest
    container_name: blackbox-exporter
    restart: always
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox-exporter/:/config
    command: --config.file=/config/blackbox.yml
    depends_on:
      - prometheus
```

### 2.2 配置文件

```yaml
modules:
  http_2xx:
    prober: http
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp 
  http_2xx_java: # 自定义状态码200的匹配规则
    prober: http
    http:
      method: GET
      fail_if_body_not_matches_regexp:
      - '"status":"UP"'
```

### 2.3 采集指标

1. 状态码：`probe_http_status_code`
2. ping延时：`probe_icmp_duration_seconds`
3. tcp连接：`probe_success`

## 3.jmx-exporter

> 目前只用来监控kafka实例

### 3.1 部署方式

**本地部署**  [[下载链接](https://github.com/prometheus/jmx_exporter/tree/release-0.20.0)](https://github.com/prometheus/jmx_exporter/tree/release-0.20.0)

监控kafka批量启动脚本：`bash jmx_kafka.sh status`

```shell
#!/bin/bash

# 定义一个函数来处理不同的 jmx 实例
manage_jmx_instance() {
  local instance="$1"
  local port=$((9800 + ${instance#kafka})) # 提取实例名称后的数字并加9800作为端口
  local pid_file="/var/run/jmx_${instance}_httpserver.pid"
  local log_file="/root/monitor/log/${instance}.log"

  case "$2" in
    start)
      if [ -f "$pid_file" ]; then
        echo "jmx_${instance}_httpserver is already running."
      else
        # 使用对应的端口号启动 jmx_prometheus_httpserver
        nohup java -jar /root/monitor/jmx_prometheus_httpserver-0.20.0.jar $port /root/monitor/${instance}.yml >> "$log_file" 2>&1 &
        echo $! > "$pid_file"
        echo "jmx_${instance}_httpserver on port $port started."
      fi
      ;;
    status)
      if [ -f "$pid_file" ]; then
        PID=$(cat "$pid_file")
        ps -p $PID > /dev/null 2>&1
        if [ $? -eq 0 ]; then
          echo "jmx_${instance}_httpserver is running (PID $PID)."
        else
          echo "jmx_${instance}_httpserver is not running but PID file exists."
        fi
      else
        echo "jmx_${instance}_httpserver is not running."
      fi
      ;;
    stop)
      if [ -f "$pid_file" ]; then
        PID=$(cat "$pid_file")
        # 使用 kill 而不是 kill -9 以避免过于激进地终止进程
        kill $PID
        rm -f "$pid_file"
        echo "jmx_${instance}_httpserver stopped."
      else
        echo "jmx_${instance}_httpserver is not running."
      fi
      ;;
    *)
      echo "Usage: $0 {start|status|stop}"
      exit 1
      ;;
  esac
}

# 根据提供的选项调用函数来管理不同的 jmx 实例,同目录下存放了三个kafka1-3.yml配置文件供exporter使用
for instance in kafka1 kafka2 kafka3; do
  manage_jmx_instance "$instance" "$1"
done

exit 0
```

### 3.2 配置文件

`kafka1.yml`

```yaml
lowercaseOutputName: true
hostPort: kafka_ip:9999
rules:
# Special cases and very specific rules
- pattern : kafka.server<type=(.+), name=(.+), clientId=(.+), topic=(.+), partition=(.*)><>Value
  name: kafka_server_$1_$2
  type: GAUGE
  labels:
    clientId: "$3"
    topic: "$4"
    partition: "$5"
- pattern : kafka.server<type=(.+), name=(.+), clientId=(.+), brokerHost=(.+), brokerPort=(.+)><>Value
  name: kafka_server_$1_$2
  type: GAUGE
  labels:
    clientId: "$3"
    broker: "$4:$5"
- pattern : kafka.coordinator.(\w+)<type=(.+), name=(.+)><>Value
  name: kafka_coordinator_$1_$2_$3
  type: GAUGE

# Generic per-second counters with 0-2 key/value pairs
- pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*, (.+)=(.+), (.+)=(.+)><>Count
  name: kafka_$1_$2_$3_total
  type: COUNTER
  labels:
    "$4": "$5"
    "$6": "$7"
- pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*, (.+)=(.+)><>Count
  name: kafka_$1_$2_$3_total
  type: COUNTER
  labels:
    "$4": "$5"
- pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*><>Count
  name: kafka_$1_$2_$3_total
  type: COUNTER

# Quota specific rules
- pattern: kafka.server<type=(.+), user=(.+), client-id=(.+)><>([a-z-]+)
  name: kafka_server_quota_$4
  type: GAUGE
  labels:
    resource: "$1"
    user: "$2"
    clientId: "$3"
- pattern: kafka.server<type=(.+), client-id=(.+)><>([a-z-]+)
  name: kafka_server_quota_$3
  type: GAUGE
  labels:
    resource: "$1"
    clientId: "$2"
- pattern: kafka.server<type=(.+), user=(.+)><>([a-z-]+)
  name: kafka_server_quota_$3
  type: GAUGE
  labels:
    resource: "$1"
    user: "$2"

# Generic gauges with 0-2 key/value pairs
- pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+), (.+)=(.+)><>Value
  name: kafka_$1_$2_$3
  type: GAUGE
  labels:
    "$4": "$5"
    "$6": "$7"
- pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+)><>Value
  name: kafka_$1_$2_$3
  type: GAUGE
  labels:
    "$4": "$5"
- pattern: kafka.(\w+)<type=(.+), name=(.+)><>Value
  name: kafka_$1_$2_$3
  type: GAUGE

# Emulate Prometheus 'Summary' metrics for the exported 'Histogram's.
#
# Note that these are missing the '_sum' metric!
- pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+), (.+)=(.+)><>Count
  name: kafka_$1_$2_$3_count
  type: COUNTER
  labels:
    "$4": "$5"
    "$6": "$7"
- pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.*), (.+)=(.+)><>(\d+)thPercentile
  name: kafka_$1_$2_$3
  type: GAUGE
  labels:
    "$4": "$5"
    "$6": "$7"
    quantile: "0.$8"
- pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+)><>Count
  name: kafka_$1_$2_$3_count
  type: COUNTER
  labels:
    "$4": "$5"
- pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.*)><>(\d+)thPercentile
  name: kafka_$1_$2_$3
  type: GAUGE
  labels:
    "$4": "$5"
    quantile: "0.$6"
- pattern: kafka.(\w+)<type=(.+), name=(.+)><>Count
  name: kafka_$1_$2_$3_count
  type: COUNTER
- pattern: kafka.(\w+)<type=(.+), name=(.+)><>(\d+)thPercentile
  name: kafka_$1_$2_$3
  type: GAUGE
  labels:
    quantile: "0.$4"

# Generic gauges for MeanRate Percent
# Ex) kafka.server<type=KafkaRequestHandlerPool, name=RequestHandlerAvgIdlePercent><>MeanRate
- pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*><>MeanRate
  name: kafka_$1_$2_$3_percent
  type: GAUGE
- pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*><>Value
  name: kafka_$1_$2_$3_percent
  type: GAUGE
- pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*, (.+)=(.+)><>Value
  name: kafka_$1_$2_$3_percent
  type: GAUGE
  labels:
    "$4": "$5"
```

### 3.3 采集指标

1. 指定kafka-topic消息数量：`round(increase(kafka_server_brokertopicmetrics_messagesin_total{topic="kafka_topic"}[1h]),0.01)`

## 4.mysql-exporter

### 4.1 部署方式

```shell
docker run -d --name mysql-exporter --restart always -p 9104:9104 -e DATA_SOURCE_NAME="user:passwd@(ip:3306)/" prom/mysqld-exporter
```

### 4.2 指标名称

1. Mysql监控agent存活：mysql_up
2. 连接数：mysql_global_status_threads_connected
3. 文件打开数：mysql_global_status_innodb_num_open_files
4. 从库只读：mysql_global_variables_read_only
5. 主从延迟：mysql_slave_status_seconds_behind_master
6. sql线程：mysql_slave_status_slave_sql_running
7. IO线程：mysql_slave_status_slave_io_running
8. 入口流量：mysql_global_status_bytes_received
9. 出口流量：mysql_global_status_bytes_sent
10. 写操作速率：mysql_global_status_commands_total
11. 慢查询：mysql_global_status_slow_queries
12. 查询速率：mysql_global_status_questions
13. 可用连接数：mysql_global_variables_max_connections
14. 缓冲池利用率：mysql_global_status_buffer_pool_pages
15. 打开文件数量限制：mysql_global_variables_open_files_limit
16. 启用了InnoDB强制恢复功能：mysql_global_variables_innodb_force_recovery
17. InnoDB日志文件大小：mysql_global_variables_innodb_log_file_size
18. InnoDB插件已启用：mysql_global_variables_ignore_builtin_innodb
19. Binary Log状态：mysql_global_variables_log_bin
20. Binlog缓存大小：mysql_global_variables_binlog_cache_size
21. 同步Binlog状态：mysql_global_variables_sync_binlog
22. IO线程状态：mysql_slave_status_slave_io_running
23. sql线程状态：mysql_slave_status_slave_sql_running
24. InnoDB日志等待量：mysql_global_status_innodb_log_waits （不为0的话，InnoDB log buffer因空间不足而等待）
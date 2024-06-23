
**完整docker-compose.yml**

```yaml
networks:
  monitor:
    name: monitor
    driver: bridge

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - 9090:9090
    restart: always
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/:/etc/prometheus
    networks:
      - monitor
    container_name: prometheus
    environment:
      - --web.enable-admin-api
      - --web.enable-lifecycle

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

  alertmanager:
      image: prom/alertmanager:latest
      container_name: alertmanager
      restart: always
      volumes:
          - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      ports:
          - "9093:9093"
      networks:
          - monitor

  dingtalk-webhook:
    image: timonwong/prometheus-webhook-dingtalk:latest
    restart: always
    volumes:
      - ./dingtalk-webhook/config.yml:/etc/prometheus-webhook-dingtalk/config.yml
      - ./dingtalk-webhook/prometheus-webhook-dingtalk:/prometheus-webhook-dingtalk
      - ./dingtalk-webhook/templates:/etc/prometheus-webhook-dingtalk/templates
    ports:
      - 8060:8060
    command: "--web.enable-ui"
    networks:
      - monitor
    container_name: dingtalk-webhook
    depends_on:
      - prometheus
      - node-exporter
      - blackbox-exporter

  grafana:
    image: grafana/grafana-enterprise
    user: root
    restart: always
    ports:
      - 3000:3000
    networks:
      - monitor
    volumes:
      - ./grafana/data:/var/lib/grafana
    container_name: grafana
    depends_on:
      - prometheus
```

## 1.Prometheus

> Prometheus只说明进行自我修改的部分，基本的安装步骤不再赘述！

### 1.1 部署方式

**本机部署**

二进制文件[[下载链接](https://prometheus.io/download/)](https://prometheus.io/download/)

```shell
# prometheus-2.53.0.linux-amd64.tar.gz 对下载后的二进制文件进行解压

# 首次启动命令
nohup ./prometheus --config.file=prometheus.yml --storage.tsdb.retention.time=90d --web.enable-lifecycle &

# check.sh -> 二进制文件部署方式的热重载脚本
#========================================
#!/bin/bash
# 定义Prometheus配置文件路径
prometheus_config="/home/sonkwo/monitor/prometheus/prometheus.yml"
# 检查配置文件
check_result=$(./promtool check config "$prometheus_config" 2>&1)
if [[ $check_result =~ "FAILED" ]]; then
  # 配置文件检查失败，输出错误信息
  echo "Prometheus config check failed!"
  echo "$check_result"
  exit 1
else
  # 配置文件检查成功，执行重载
  curl -X POST http://localhost:9090/-/reload
  echo "Prometheus config check successful!"
fi
#========================================
```

**systemd部署**

```shell
# 可选添加用户
sudo useradd --no-create-home --shell /bin/false prometheus
# prometheus.service 文件

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/usr/share/prometheus/consoles \
  --web.console.libraries=/usr/share/prometheus/console_libraries

[Install]
WantedBy=multi-user.target

# 进行对应的初次重载命令
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

**docker-compose部署**

```yaml
networks:
  monitor:
    name: monitor
    driver: bridge

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - 9090:9090
    restart: always
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/:/etc/prometheus
    networks:
      - monitor
    container_name: prometheus
    environment:
      - --web.enable-admin-api
      - --web.enable-lifecycle
```

### 1.2 主配置文件

> 采用Git+Jenkins(CI/CD)+Docker方式，通过本地编辑 -> 推送Git -> CI/CD执行拉取和热更新 -> 完成配置文件修改步骤

`prometheus.yml`

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
# 可以为通配符路径，注意是宿主机路径还是容器路径
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
  - ./rules/*.yml

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    metrics_path: /metrics
    static_configs:
    file_sd_configs:
      - files: ['./node-exporter/*.yml']
        refresh_interval: 15s
    relabel_configs:
      - source_labels: [__address__]
        regex: (.*)
        target_label: __address__
        replacement: $1

  - job_name: "blackbox_http_2xx"
    scrape_interval: 90s
    metrics_path: /probe
    params:
      module:
      - http_2xx
    static_configs:
    file_sd_configs:
      - files: ['./blackbox_http_2xx/*.yml']
        refresh_interval: 15s
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115  # blackbox_exporter所在的机器和端口

  - job_name: "blackbox_http_3min"
    scrape_interval: 3m # 可以单独调整blackbox的抓取频率
    metrics_path: /probe
    params:
      module:
      - http_2xx
    static_configs:
    file_sd_configs:
      - files: ['./blackbox_http_3min/*.yml']
        refresh_interval: 15s
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115  # blackbox_exporter所在的机器和端口

  - job_name: "blackbox_tcp_connect"
    metrics_path: /probe
    params:
      module:
      - tcp_connect
    static_configs:
    file_sd_configs:
      - files: ['./blackbox_tcp_connect/*.yml']
        refresh_interval: 15s
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115  # blackbox_exporter所在的机器和端口

  - job_name: "blackbox_ping"
    metrics_path: /probe
    params:
      module:
      - icmp
    static_configs:
    file_sd_configs:
      - files: ['./blackbox_ping/*.yml']
        refresh_interval: 15s
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115  # blackbox_exporter所在的机器和端口

# 以下为大数据Hadoop所需的exporter
  - job_name: "fsimage"
    scrape_interval: 1m   # Depends on how often the name node writes a fsimage file.
    scrape_timeout: 40s    # Depends on size
    static_configs:
      - targets: [ 'xxxx:9709' ]
        labels:
          project: fsimage

  - job_name: "kafka-exporter"
    scrape_interval: 20s
    scrape_timeout: 15s
    static_configs:
      - targets: ['xxxx:9308']
        labels:
          project: kafka-exporter
          instance: xxxx:9308

  - job_name: "jmx-exporter"
    metrics_path: /metrics
    static_configs:
    file_sd_configs:
      - files: ['./jmx-exporter/*.yml']
        refresh_interval: 15s
```

### 1.3 从配置文件

`node-exporter.yml`

```yaml
- targets:
  - xxxx:9100
  labels:    
    project: configure by ur self
    instance: xxx:9100
    name: configure by ur self
```

`blackbox-exporter.yml`

```yaml
- targets:
  - put a url on this line, include https or http
  labels:
    project: configure by ur self
    service: xxx
```

`jmx-exporter.yml`

```yaml
- targets:
    - kafka:9803
  labels:
    project: node-exporter
    instance: kafka:9803
    process: kafka
```

`node-rules.yml`

```yaml
groups:
  - name: 服务器状态监控
    rules:
      - alert: CPU 监控
        expr: round((1 - avg(rate(node_cpu_seconds_total{instance="xxxxx:9100",mode="idle"}[1m])) by (instance)) * 100, 0.01) > 80
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: 服务器CPU占用率高
          description: "{{$labels.instance}}服务器 CPU 已使用: {{$value}}%"
      - alert: 内存监控
        expr: round((1 - (node_memory_MemAvailable_bytes{instance="xxxxx:9100"} / (node_memory_MemTotal_bytes{instance="xxxxx:9100"})))* 100,0.01) > 95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: 服务器内存占用过高
          description: "{{$labels.instance}}服务器内存已使用: {{$value}}%"
      - alert: 磁盘监控
        expr: round((node_filesystem_size_bytes{instance="xxxxx:9100",mountpoint="/"}-node_filesystem_free_bytes{instance="xxxxx:9100",mountpoint="/"})*100/(node_filesystem_avail_bytes{instance="xxxxx:9100",mountpoint="/"}+(node_filesystem_size_bytes{instance="xxxxx:9100",mountpoint="/"}-node_filesystem_free_bytes{instance="xxxxx:9100",mountpoint="/"})),0.01) > 90
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: 服务器磁盘占用过高
          description: "{{$labels.instance}}服务器磁盘已使用: {{$value}}%"
      - alert: 网络连接数监控
        expr: node_netstat_Tcp_CurrEstab{instance="xxxxx:9100"} > 500
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: 服务器网络连接数过高
          description: "{{$labels.instance}}服务器当前网络连接数: {{$value}}"
```

`blackbox-rules.yml`

```yaml
groups:
- name: hosts监控
  rules:
  - alert: 状态码监控(monitor)
    expr: probe_http_status_code{service="xxx"} != 200
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: xxx状态码不为200(monitor)
      description: '检测服务: {{$labels.service}}, 状态码为: {{$value}}'
```

## 2.Grafana

> Grafana不在赘述配置dashboard等基础环节，只写一些流传较为少的配置方法

### 2.1 部署方式

**docker-compose部署**

*在执行docker compose up -d之前，需要先执行docker run的步骤将想要持久化的容器内文件通过docker cp拷贝到宿主机上！*

```yaml
  grafana:
    image: grafana/grafana-enterprise
    user: root
    restart: always
    ports:
      - 3000:3000
    networks:
      - monitor
    volumes:
      - ./grafana/data:/var/lib/grafana
      - ./grafana/plugins:/var/lib/grafana/plugins
      - ./grafana/log:/var/log/grafana
      - ./grafana/conf:/etc/grafana
    container_name: grafana
    depends_on:
      - prometheus
    extra_hosts:
      - "url:ip_address"
```

### 2.2 连接LDAP

**设置配置文件中的auth.ldap为true  文件路径：`/usr/share/grafana/conf/defaults.ini`**

```ini
[auth.ldap]
# Set to `true` to enable LDAP integration (default: `false`)
enabled = true

# Path to the LDAP specific configuration file (default: `/etc/grafana/ldap.toml`)
config_file = /etc/grafana/ldap.toml

# Allow sign-up should be `true` (default) to allow Grafana to create users on successful LDAP authentication.
# If set to `false` only already existing Grafana users will be able to login.
allow_sign_up = true
```

**添加ldap相关的设置   文件路径：`/etc/grafana/ldap.toml`**

```toml
# To troubleshoot and get more log info enable ldap debug logging in grafana.ini
# [log]
# filters = ldap:debug
[[servers]]
# Ldap server host (specify multiple hosts space separated)
host = "xxxxx"
# Default port is 389 or 636 if use_ssl = true
port = 636
# Set to true if LDAP server should use an encrypted TLS connection (either with STARTTLS or LDAPS)
use_ssl = true
# If set to true, use LDAP with STARTTLS instead of LDAPS
start_tls = false
# set to true if you want to skip ssl cert validation
ssl_skip_verify = false
# set to the path to your root CA certificate or leave unset to use system defaults
# root_ca_cert = "/path/to/certificate.crt"
# Authentication against LDAP servers requiring client certificates
# client_cert = "/path/to/client.crt"
# client_key = "/path/to/client.key"

# Search user bind dn
bind_dn = "uid=xxxxx,cn=users,cn=accounts,dc=xxxxxxx,dc=cn"
# Search user bind password
# If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
bind_password = """"""

# Timeout in seconds (applies to each host specified in the 'host' entry (space separated))
timeout = 10

# User search filter, for example "(cn=%s)" or "(sAMAccountName=%s)" or "(uid=%s)"
search_filter = "(uid=%s)"

# An array of base dns to search through
search_base_dns = ["cn=users,cn=accounts,dc=sonkwo,dc=cn"]

## For Posix or LDAP setups that does not support member_of attribute you can define the below settings
## Please check grafana LDAP docs for examples
#group_search_filter = "(&(objectClass=groupOfNames)(memberOf=%s))"
#group_search_base_dns = ["cn=groups,cn=accounts,dc=sonkwo,dc=cn"]
#group_search_filter_user_attribute = "uid"

# Specify names of the ldap attributes your ldap uses
[servers.attributes]
name = "givenName"
surname = "uid"
username = "uid"
member_of = "memberOf"
email =  "mail"

# Map ldap groups to grafana org roles
[[servers.group_mappings]]
group_dn = "cn=admins,cn=groups,cn=accounts,dc=xxxxx,dc=cn"
org_role = "Admin"
# To make user an instance admin  (Grafana Admin) uncomment line below
grafana_admin = true
# The Grafana organization database id, optional, if left out the default org (id 1) will be used
# org_id = 1

[[servers.group_mappings]]
group_dn = "cn=users,cn=groups,cn=accounts,dc=xxxxx,dc=cn"
org_role = "Editor"

#[[servers.group_mappings]]
# If you want to match all (or no ldap groups) then you can use wildcard
#group_dn = "*"
#org_role = "Viewer"
```
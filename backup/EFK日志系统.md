**注：现在新版本elastic stack已经支持将kafka作为输出和输入的目标**

当日志不是结构化数据：*.log->filebeat->kafka->logstash->elasticsearch

当日志是结构化数据：*.log->filebeat->kafka->filebeat->elasticsearch

区别在于是否需要logstash进行日志的过滤和结构化；

## 结构化日志

**连接示例配置文件**

### filebeat 1

> 此filebeat实例为日志采集端，即kafka生产者，可以有多个实例进行采集

```yaml
filebeat.inputs:
- type: filestream #新版本常用文件流输入方式
  id: my-filestream-id #每个文件流输入必须有一个唯一的 ID。省略或更改文件流 ID 可能会导致数据重复。如果没有唯一的 ID，filestream 无法正确跟踪文件的状态。
  paths:
    - /var/log/messages
    - /var/log/*.log
    
- type: filestream 
  id: apache-filestream-id
  paths:
    - "/var/log/apache2/*"
  fields:
    apache: true #fields字段随后会被存储在es中通过kibana进行查询，便于识别这些日志数据来源于Apache服务；
    
    
#输出到kafka 此输出可以连接到 Kafka 0.8.2.0 及更高版本
output.kafka:
  #用于读取集群元数据的brokers
  hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]
  #消息主题选择+分区
  topic: '%{[fields.log_topic]}'
  partition.round_robin:
    reachable_only: false

  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000 #大于 max_message_bytes 的事件将被丢弃
```

### kafka+kafka-ui

> kafka为docker部署，镜像版本为：apache/kafka:3.7.0

**docker-compose.yml**

```yaml
networks:
  kafka:
    name: kafka
    driver: bridge

services:
  kafka:
    image: apache/kafka:3.7.0
    restart: unless-stopped
    user: root
    volumes:
      - ./config:/mnt/shared/config	# 需要看kafka文档，需要挂载此路径下的配置文件才可以进行更新
      - ./data:/var/lib/kafka/data
      - ./logs:/opt/kafka/logs
      - ./kraft-combined-logs:/var/log/kafka/kraft-combined-logs
    ports:
      - "9092:9092"
      - "9093:9093"
      - "9094:9094"
    container_name: kafka
    networks:
      - kafka

  kafka-ui:
    container_name: kafka-ui
    restart: unless-stopped
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8001:8080"
    environment:
      DYNAMIC_CONFIG_ENABLED: true
    volumes:
      - ./kafka-ui/dynamic_config.yaml:/etc/kafkaui/dynamic_config.yaml
    deploy:
      resources:
        limits:
          memory: 1G
    networks:
      - kafka
```



> 只列出示例配置文件中的原始部分和修改部分

`server.properties`

```shell
#listeners=PLAINTEXT://:9092,CONTROLLER://:9093
listeners=INSIDE://0.0.0.0:9092,OUTSIDE://0.0.0.0:9094,CONTROLLER://:9093

# Name of listener used for communication between brokers.
#inter.broker.listener.name=PLAINTEXT,INSIDE,OUTSIDE
inter.broker.listener.name=INSIDE

# Listener name, hostname and port the broker will advertise to clients.
# If not set, it uses the value for "listeners".
advertised.listeners=INSIDE://moniti.xxxxxx.cn:9092,OUTSIDE://monito.xxxxxx.cn:9094

# A comma-separated list of the names of the listeners used by the controller.
# If no explicit mapping set in `listener.security.protocol.map`, default will be using PLAINTEXT protocol
# This is required if running in KRaft mode.
controller.listener.names=CONTROLLER

# Maps listener names to security protocols, the default is for them to be the same. See the config documentation for more details
#listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
listener.security.protocol.map=INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT,CONTROLLER:PLAINTEXT

# A comma separated list of directories under which to store log files
log.dirs=/var/log/kafka/kraft-combined-logs

# The minimum age of a log file to be eligible for deletion due to age
log.retention.hours=168
```

`dynamic_config.yaml`*——此文件为kafka-ui自动生成*

```yaml
auth:
  type: DISABLED #此位置可以接入ldap
kafka:
  clusters:
  - bootstrapServers: kafka:9092
    name: kafka-log
    properties: {}
    readOnly: false
rbac:
  roles: []
webclient: {}
```



### filebeat 2

> 此filebeat为消费者，消费kafka中的日志信息并由processer处理后存入es对应的index中

`filebeat.docker.yml`

```yaml
filebeat.inputs:
- type: kafka
  hosts:  # Kafka 引导主机（代理）列表
    - kafka-broker-1:9092
    - kafka-broker-2:9092
  topics: ["my-topic"]
  group_id: "filebeat"  #消费者组 ID
  fetch.min: 500000   #要等待的最小字节数,默认为 1。

processors:
  - decode_json_fields:
      fields: ["message"]
      max_depth: 2
      target: ""
      overwrite_keys: true
      add_error_key: true
  
# 输出到es
output.elasticsearch:
  hosts: ["https://myEShost:9200"]
  username: "filebeat_writer"
  password: "YOUR_PASSWORD"
  indices:
  - index: "%{[fields.log_topic]}-%{[agent.version]}-%{+yyyy.MM.dd}"
    when.contains:
      fields.log_topic: "log_name_example"
  - index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"
  
# 确保Filebeat可以使用索引模板
setup.ilm.enabled: false
setup.template.enabled: false
setup.template.overwrite: true
setup.template.json.enabled: true
setup.template.name: "filebeat-%{[agent.version]}"
setup.template.pattern: "filebeat-%{[agent.version]}-*"
setup.template.settings:
  index.number_of_shards: 1
  index.number_of_replicas: 0
```

### Elasticsearch

`elasticsearch.yml`

```yaml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
discovery.seed_hosts: ["192.168.8.99"]
#cluster.initial_master_nodes: ["docker-cluster"]
discovery.type: single-node
# 开启x-pack插件,用于添加账号密码、安全控制等配置
xpack.security.enabled: true
#xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/data/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/data/certs/elastic-certificates.p12
#xpack.security.enrollment.enabled: true
```

### Kibana

`kibana.yml`

```yaml
# Default Kibana configuration for docker target
server.host: "0.0.0.0"
server.shutdownTimeout: "5s"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
monitoring.ui.container.elasticsearch.enabled: true
elasticsearch.ssl.certificateAuthorities: "/usr/share/kibana/config/certs/elastic-stack-ca.p12"
elasticsearch.username: "kibana"
elasticsearch.password: "elastic"
i18n.locale: "zh-CN"
```

### Elastalert

[[官方文档](https://elastalert.readthedocs.io/en/latest/index.html)](https://elastalert.readthedocs.io/en/latest/index.html)

#### 配置文件

`config.yaml`

```yaml
# This is the folder that contains the rule yaml files
# This can also be a list of directories
# Any .yaml file will be loaded as a rule
rules_folder: /opt/elastalert/rules

# How often ElastAlert will query Elasticsearch
# The unit can be anything from weeks to seconds
run_every:
  minutes: 1

# ElastAlert will buffer results from the most recent
# period of time, in case some log sources are not in real time
buffer_time:
  minutes: 15

# The Elasticsearch hostname for metadata writeback
# Note that every rule can have its own Elasticsearch host
es_host: 10.1.1.30

# The Elasticsearch port
es_port: 9200

# The AWS region to use. Set this when using AWS-managed elasticsearch
#aws_region: us-east-1

# The AWS profile to use. Use this if you are using an aws-cli profile.
# See http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
# for details
#profile: test

# Optional URL prefix for Elasticsearch
#es_url_prefix: elasticsearch

# Optional prefix for statsd metrics
#statsd_instance_tag: elastalert

# Optional statsd host
#statsd_host: dogstatsd

# Connect with TLS to Elasticsearch
#use_ssl: True

# Verify TLS certificates
#verify_certs: True

# Show TLS or certificate related warnings
#ssl_show_warn: True

# GET request with body is the default option for Elasticsearch.
# If it fails for some reason, you can pass 'GET', 'POST' or 'source'.
# See https://elasticsearch-py.readthedocs.io/en/master/connection.html?highlight=send_get_body_as#transport
# for details
#es_send_get_body_as: GET

# Option basic-auth username and password for Elasticsearch
es_username: elastic
es_password: elastic

# Use SSL authentication with client certificates client_cert must be
# a pem file containing both cert and key for client
#ca_certs: /path/to/cacert.pem
#client_cert: /path/to/client_cert.pem
#client_key: /path/to/client_key.key

# The index on es_host which is used for metadata storage
# This can be a unmapped index, but it is recommended that you run
# elastalert-create-index to set a mapping
writeback_index: elastalert_status

# If an alert fails for some reason, ElastAlert will retry
# sending the alert until this time period has elapsed
alert_time_limit:
  days: 2

# 可选的时间戳格式。
# ElastAlert将使用此格式在警报消息和日志消息中打印时间戳。
custom_pretty_ts_format: '%Y-%m-%d %H:%M'

# 自定义日志配置
# 如果您希望设置自己的日志配置以记录到文件中，或记录到Logstash，并且/或者修改日志级别，请使用下面的配置并根据需要进行调整。
# 注意：如果您以--verbose/--debug运行ElastAlert，“elastalert”记录器的日志级别将更改为INFO，如果它还不是INFO/DEBUG。
logging:
 version: 1
 incremental: false
 disable_existing_loggers: false
 formatters:
   logline:
     format: '%(asctime)s %(levelname)+8s %(name)+20s %(message)s'

 handlers:
   console:
     class: logging.StreamHandler
     formatter: logline
     level: DEBUG
     stream: ext://sys.stderr

   file:
     class : logging.FileHandler
     formatter: logline
     level: DEBUG
     filename: elastalert.log

 loggers:
   elastalert:
     level: WARN
     handlers: []
     propagate: true

   elasticsearch:
     level: WARN
     handlers: []
     propagate: true

   elasticsearch.trace:
     level: WARN
     handlers: []
     propagate: true

   '':  # 根记录器
     level: WARN
     handlers:
       - console
       - file
     propagate: false
```

`nginx-rules.yaml`

```yaml
name: "Nginx-Error"
type: "frequency"
index: "nginx-error-8.13.2*"  # 查询日志所在的索引
is_enabled: true
num_events: 3  # 事件出现的次数
timeframe:
  minutes: 2   # 1分钟内出现了num_events几次就报警
realert:
  minutes: 180   # 5分钟内忽略重复告警
timestamp_field: "@timestamp"
timestamp_type: "iso"
use_strftime_index: true
filter:
  - query:
      query_string:
        query: "message:\"error\" NOT message:\"No such file or directory\" NOT message:\"forbidden by rule\""  # 告警查询语句

alert_text_type: alert_text_only
alert_text: "### Nginx-Error(Prod)日志告警\n- **采集位置:** {0}\n- **文件:** {1}\n- **匹配次数:** {2}\n- **日志id:** {3}\n\n- [点我去查询]()"
# 上述括号中填写查询链接
alert_text_args:   # 告警模板中用到的参数
  - agent.name
  - "log.file.path"
  - num_matches
  - _id
  - _id

alert:
  - "dingtalk"
dingtalk_access_token: ""
dingtalk_msgtype: "markdown"
```

#### Enhancements自定义开发&镜像构建

> [!TIP]
>
> 初始化流程可以按照官方[[文档](https://elastalert.readthedocs.io/en/latest/recipes/adding_enhancements.html)](https://elastalert.readthedocs.io/en/latest/recipes/adding_enhancements.html)进行

Example：Ruby日志格式匹配提取request-id

创建文件夹

```shell
mkdir elastalert/elastalert_modules
touch __init__.py extract_request_id.py
```

`extract_request_id.py`

```python
from elastalert.enhancements import BaseEnhancement
import re

class Extract_Requestid(BaseEnhancement):
    def process(self, match):
        # 定义正则表达式模式
        pattern = r"\b[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}\b"
        # 检查 'message' 字段是否存在于match中
        if 'message' in match:
            log_message = match['message']
            match_obj = re.search(pattern, log_message)
            if match_obj:
                match['extract_requestid'] = match_obj.group(0)
            else:
                match['extract_requestid'] = "request id not found"

```

### docker compose 部署

#### 生产端

```yaml
services:
  filebeat:
    image: elastic/filebeat:8.13.2
    container_name: filebeat
    user: root
    restart: unless-stopped
    volumes:
      - ./filebeat.docker.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/usr/share/filebeat/data:rw
      - /home:/home
      - /var/log:/var/log
```

#### 消费端

```yaml
networks:
  logstack:
    name: logstack
    driver: bridge

services:
  filebeat:
    image: elastic/filebeat:8.13.2
    container_name: filebeat
    restart: unless-stopped
    ports:
      - "5067:5067"
    volumes:
      - ./filebeat/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./filebeat/data:/usr/share/filebeat/data:rw
    networks:
      - logstack

  elasticsearch:
    image: elastic/elasticsearch:8.13.2
    container_name: elasticsearch
    restart: unless-stopped
    environment:
      - "ES_JAVA_OPTS=-Xms6144m -Xmx6144m"
      - "cluster.name=elasticsearch"
    volumes:
      - ./elasticsearch/data:/usr/share/elasticsearch/data
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - logstack

  kibana:
    image: elastic/kibana:8.13.2
    container_name: kibana
    restart: unless-stopped
    volumes:
      - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml
      - ./kibana/certs:/usr/share/kibana/config/certs
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - logstack

  metricbeat:
    image: elastic/metricbeat:8.13.2
    container_name: metricbeat
    restart: unless-stopped
    volumes:
      - ./metricbeat/metricbeat.docker.yml:/usr/share/metricbeat/metricbeat.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro
      - /proc:/hostfs/proc:ro
      - /:/hostfs:ro
    networks:
      - logstack

  esalert:
    image: jertel/elastalert2:2.18.0
    container_name: esalert
    restart: unless-stopped
    user: root
    volumes:
      - ./esalert/config.yaml:/opt/elastalert/config.yaml
      - ./esalert/rules:/opt/elastalert/rules
    depends_on:
      - elasticsearch
    networks:
      - logstack
    environment:
      TZ: "Asia/Shanghai"
```



## 非结构化日志(未测试)

### filebeat 1

```yaml
filebeat.inputs:
- type: filestream #新版本常用文件流输入方式
  id: my-filestream-id #每个文件流输入必须有一个唯一的 ID。省略或更改文件流 ID 可能会导致数据重复。如果没有唯一的 ID，filestream 无法正确跟踪文件的状态。
  paths:
    - /var/log/messages
    - /var/log/*.log
    
- type: filestream 
  id: apache-filestream-id
  paths:
    - "/var/log/apache2/*"
  fields:
    apache: true #fields字段随后会被存储在es中通过kibana进行查询，便于识别这些日志数据来源于Apache服务；
    
    
#输出到kafka 此输出可以连接到 Kafka 0.8.2.0 及更高版本
output.kafka:
  #用于读取集群元数据的brokers
  hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]
  #消息主题选择+分区
  topic: '%{[fields.log_topic]}'
  partition.round_robin:
    reachable_only: false

  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000 #大于 max_message_bytes 的事件将被丢弃
```

### kafka

```yaml
#暂定
```

### logstash

```conf
input {
  kafka {
    id => "my-kafka-consumer" # 唯一标识此Kafka消费者
    topics => ["my-topic"] # Kafka主题列表
    topic_id => "%{topic}" # 用于标识消息来自哪个主题
    group_id => "my-consumer-group" # Kafka消费者组ID
    bootstrap_servers => "localhost:9092" # Kafka服务器地址
    # 其他可选配置
    auto_offset_reset => "earliest" # 当没有初始偏移量时，从最早的消息开始消费
    enable_auto_commit => true # 自动提交偏移量
    max_poll_records => 500 # 每次轮询获取的最大记录数
    max_poll_interval_ms => 300000 # 轮询间隔的最大毫秒数
  }
}

filter {
  # 在这里添加任何你需要的过滤或数据转换操作
  # 例如，使用grok插件解析日志格式
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{WORD:thread} %{GREEDYDATA:message}" }
  }
  date {
    match => ["timestamp", "ISO8601"] # 转换时间戳格式
  }
  mutate {
    lowercase => ["level"] # 将日志级别转换为小写
  }
}

output {
  elasticsearch {
    hosts => ["https://...] 
    cacert => '/etc/logstash/config/certs/ca.crt' 
  }
}
```

# ES索引生命周期

## 以下面的顺序来配置

### Kibana

1. 点击管理->索引生命周期策略->创建策略，选择需要的阶段和在多少天后删除该数据；

2. 点击管理->索引管理->索引模板->创建模板：

   - 名称修改完貌似是不可重新编辑的；

   - 索引模式可以以通配符'*'的方式来匹配需要的索引，首次建议使用一个进行测试；

   - 数据流不要创建；

   - 打开下面的**允许自动创建**；

   - **索引设置**可以对索引生命周期策略进行关联：

     ```json
     {
       "index": {
         "lifecycle": {
           "name": "log-rotate"
         }
       }
     }
     ```

   - **映射**可以从已经存在的索引当中复制json到这里粘贴进行编辑；

### Filebeat

> 这里的filebeat为消费kafka后将日志消息存储到Elasticsearch中的filebeat实例，主要看配置文件变化

```yaml
# 按照官方文档中开启对应的配置就可以
setup.ilm.enabled: true
setup.ilm.policy_name: "log-rotate"
setup.template.enabled: false
setup.template.overwrite: true
setup.template.json.enabled: true
setup.template.name: "filebeat-%{[agent.version]}"
setup.template.pattern: "filebeat-%{[agent.version]}-*"
setup.template.settings:
  index.number_of_shards: 1
  index.number_of_replicas: 0
```

# 

# 问题

1. this action would add [2] shards, but this cluster currently has [1000]/[1000] maximum normal shards open：es分片数量超限

   ```
   PUT /_cluster/settings
   {
     "persistent": {
       "cluster": {
         "max_shards_per_node":10000
       }
     }
   }
   
   #查询
   GET /_cluster/settings?pretty
   ```
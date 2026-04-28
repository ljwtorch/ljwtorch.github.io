## 1.docker部署

### 1.1 以banyandb为存储(基础配置无消息队列)

- banyandb web页面：http://ip:17913/banyandb/dashboard
- 注：在接入12个java服务，测试环境存储24小时指标的资源消耗：
  - oap-server：4G内存
  - banyandb：1G内存，1.1G硬盘空间
  - ui：800M内存

`docker-compose.yml`

```yaml
services:
  banyandb:
    image: registry.cn-beijing.aliyuncs.com/witter/skywalking-banyandb:0.8.0
    container_name: banyandb
    restart: always
    ports:
      - "17912:17912"
      - "17913:17913"
    volumes:
      - ./banyandb/data:/var/lib/banyandb
    environment:
      - BANYANDB_JAVA_OPTS=-Xms1g -Xmx1g
    command: standalone --stream-root-path /var/lib/banyandb/stream-data --measure-root-path /var/lib/banyandb/measure-data --observability-modes=native
    healthcheck:
      test: [ "CMD", "sh", "-c", "nc -nz 127.0.0.1 17912" ]
      interval: 5s
      timeout: 60s
      retries: 120
    networks:
      - skywalking

  oap:
    image: registry.cn-beijing.aliyuncs.com/witter/skywalking-oap-server:10.2.0
    container_name: oap-server
    restart: always
    depends_on:
      banyandb:
        condition: service_healthy
    ports:
      - "11800:11800"
      - "12800:12800"
    # volumes:
      # - ./oap/config/application.yml:/skywalking/config/application.yml
    environment:
      - SW_STORAGE=banyandb
      - SW_STORAGE_BANYANDB_TARGETS=banyandb:17912
      - SW_TELEMETRY=prometheus
      - JAVA_OPTS=-Xms2048m -Xmx2048m
    networks:
      - skywalking

  ui:
    image: registry.cn-beijing.aliyuncs.com/witter/skywalking-ui:10.2.0
    container_name: ui
    restart: always
    depends_on:
      - oap
    ports:
      - "8081:8080"
    environment:
      - SW_OAP_ADDRESS=http://oap:12800
      - SW_ZIPKIN_ADDRESS=http://oap:9412
    networks:
      - skywalking

networks:
  skywalking:
    driver: bridge
```

### 1.2 以elasticsearch为存储，skywalking-satellite中转；

> 1. SkyWalking Satellite 目前只支持把 Logs 转发到 Kafka（插件native-log-kafka-forwarder）；
> 2. Tracing 和 Metrics 没有对应的 Kafka forwarder，只能走 gRPC 转发到 OAP；

推荐场景是：**Java Agent → Satellite（11801）→ OAP → Elasticsearch**

#### 1.2.1 satellite

`satellite_config.yaml`

```yaml
# 日志记录器配置
logger:
  # 日志格式模板配置
  log_pattern: ${SATELLITE_LOGGER_LOG_PATTERN:%time [%level][%field] - %msg}
  # 时间格式模板配置
  time_pattern: ${SATELLITE_LOGGER_TIME_PATTERN:2006-01-02 15:04:05.000}
  # 允许打印的最低日志级别
  level: ${SATELLITE_LOGGER_LEVEL:info}

# Satellite 自身的遥测（telemetry）配置
telemetry:
  # 部署的空间概念，类似于 Kubernetes 中的 namespace
  cluster: ${SATELLITE_TELEMETRY_CLUSTER:satellite-cluster}
  # 部署的服务分组概念，类似于 Kubernetes 中的 service 资源
  service: ${SATELLITE_TELEMETRY_SERVICE:satellite-service}
  # 最小运行单元，类似于 Kubernetes 中的 pod
  instance: ${SATELLITE_TELEMETRY_SERVICE:satellite-instance}
  # 遥测数据导出类型，支持 "prometheus"、"metrics_service"、"pprof" 或 "none"，多个用逗号分隔
  export_type: ${SATELLITE_TELEMETRY_EXPORT_TYPE:prometheus,pprof}

# 被不同管道（pipe）中具体插件所引用的共享插件配置
sharing:
  clients:
    - plugin_name: "grpc-client"
      # gRPC 服务器地址发现器类型
      finder_type: ${SATELLITE_GRPC_CLIENT_FINDER:static}
      # gRPC 服务器地址（默认 localhost:11800）
      server_addr: ${SATELLITE_GRPC_CLIENT:127.0.0.1:11800}
      # 是否启用 TLS
      enable_tls: ${SATELLITE_GRPC_ENABLE_TLS:false}
      # 客户端证书文件路径（启用 TLS 时生效）
      client_pem_path: ${SATELLITE_GRPC_CLIENT_PEM_PATH:"client.pem"}
      # 客户端私钥文件路径（启用 TLS 时生效）
      client_key_path: ${SATELLITE_GRPC_CLIENT_KEY_PATH:"client.key"}
      # 是否跳过服务器证书链和主机名验证
      insecure_skip_verify: ${SATELLITE_GRPC_INSECURE_SKIP_VERIFY:false}
      # CA 证书文件路径（启用 TLS 时生效）
      ca_pem_path: ${SATELLITE_grpc_CA_PEM_PATH:"ca.pem"}
      # 连接检查周期（秒）
      check_period: ${SATELLITE_GRPC_CHECK_PERIOD:5}
      # 发送请求时的认证值
      authentication: ${SATELLITE_GRPC_AUTHENTICATION:""}
      # gRPC 请求超时配置
      timeout:
        # 单次 unary 请求超时
        unary: ${SATELLITE_GRPC_TIMEOUT_UNARY:5s}
        # 流式 unary 请求超时
        stream: ${SATELLITE_GRPC_TIMEOUT_STREAM:20s}
  servers:
    - plugin_name: "grpc-server"
      # gRPC 服务器监听地址
      address: ${SATELLITE_GRPC_ADDRESS:":11800"}
      # TLS 证书文件路径
      tls_cert_file: ${SATELLITE_GRPC_TLS_CERT_FILE:""}
      # TLS 私钥文件路径
      tls_key_file: ${SATELLITE_GRPC_TLS_KEY_FILE:""}
    - plugin_name: "prometheus-server"
      # prometheus 服务器监听地址
      address: ${SATELLITE_PROMETHEUS_ADDRESS:":11234"}
      # prometheus 服务器的指标暴露路径（endpoint）
      endpoint: ${SATELLITE_PROMETHEUS_ENDPOINT:"/metrics"}

# 工作管道（pipe）配置
pipes:
  - common_config:
      pipe_name: meterpipe
    gatherer:
      server_name: "grpc-server"
      receiver:
        plugin_name: "grpc-native-meter-receiver"
      queue:
        plugin_name: "mmap-queue"
        # 最大缓冲事件数量
        event_buffer_size: ${SATELLITE_QUEUE_EVENT_BUFFER_SIZE:5000}
        # 队列分区数量
        partition: ${SATELLITE_QUEUE_PARTITION:4}
    processor:
      filters:
    sender:
      client_name: grpc-client
      forwarders:
        - plugin_name: native-meter-grpc-forwarder
          # 用于托管服务实例路由规则的 LRU 缓存大小
          routing_rule_lru_cache_size: ${SATELLITE_METERPIPE_FORWARD_ROUTING_RULE_LRU_CACHE_SIZE:5000}
          # 用于托管服务实例路由规则的 LRU 缓存 TTL（秒）
          routing_rule_lru_cache_ttl: ${SATELLITE_METERPIPE_FORWARD_ROUTING_RULE_LRU_CACHE_TTL:180}
```

#### 1.2.2 oap-server

`oap-server-application.yml`

```yaml
cluster:
  selector: ${SW_CLUSTER:standalone}
  standalone:
  # 请确保您的 ZooKeeper 版本为 3.5+，但也兼容 ZooKeeper 3.4.x。
  # 如需使用 3.4.x，请将 oap-libs 文件夹中的 ZooKeeper 3.5+ 库替换为对应的 3.4.x 库。
  zookeeper:
    namespace: ${SW_NAMESPACE:""}
    hostPort: ${SW_CLUSTER_ZK_HOST_PORT:localhost:2181}
    # 重试策略
    baseSleepTimeMs: ${SW_CLUSTER_ZK_SLEEP_TIME:1000}     # 重试之间的初始等待时间（毫秒）
    maxRetries: ${SW_CLUSTER_ZK_MAX_RETRIES:3}            # 最大重试次数
    # 是否启用 ACL
    enableACL: ${SW_ZK_ENABLE_ACL:false}                  # 默认关闭 ACL
    schema: ${SW_ZK_SCHEMA:digest}                        # 仅支持 digest 模式
    expression: ${SW_ZK_EXPRESSION:skywalking:skywalking}
    internalComHost: ${SW_CLUSTER_INTERNAL_COM_HOST:""}
    internalComPort: ${SW_CLUSTER_INTERNAL_COM_PORT:-1}
  kubernetes:
    namespace: ${SW_CLUSTER_K8S_NAMESPACE:default}
    labelSelector: ${SW_CLUSTER_K8S_LABEL:app=collector,release=skywalking}
    uidEnvName: ${SW_CLUSTER_K8S_UID:SKYWALKING_COLLECTOR_UID}
  consul:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    # Consul 集群节点，示例：10.0.0.1:8500,10.0.0.2:8500,10.0.0.3:8500
    hostPort: ${SW_CLUSTER_CONSUL_HOST_PORT:localhost:8500}
    aclToken: ${SW_CLUSTER_CONSUL_ACLTOKEN:""}
    internalComHost: ${SW_CLUSTER_INTERNAL_COM_HOST:""}
    internalComPort: ${SW_CLUSTER_INTERNAL_COM_PORT:-1}
  etcd:
    # etcd 集群节点，示例：10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379
    endpoints: ${SW_CLUSTER_ETCD_ENDPOINTS:localhost:2379}
    namespace: ${SW_CLUSTER_ETCD_NAMESPACE:/skywalking}
    serviceName: ${SW_CLUSTER_ETCD_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    authentication: ${SW_CLUSTER_ETCD_AUTHENTICATION:false}
    user: ${SW_CLUSTER_ETCD_USER:}
    password: ${SW_CLUSTER_ETCD_PASSWORD:}
    internalComHost: ${SW_CLUSTER_INTERNAL_COM_HOST:""}
    internalComPort: ${SW_CLUSTER_INTERNAL_COM_PORT:-1}
  nacos:
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    hostPort: ${SW_CLUSTER_NACOS_HOST_PORT:localhost:8848}
    # Nacos 命名空间
    namespace: ${SW_CLUSTER_NACOS_NAMESPACE:"public"}
    contextPath: ${SW_CLUSTER_NACOS_CONTEXT_PATH:""}
    # Nacos 认证用户名
    username: ${SW_CLUSTER_NACOS_USERNAME:""}
    password: ${SW_CLUSTER_NACOS_PASSWORD:""}
    # Nacos 认证 accessKey
    accessKey: ${SW_CLUSTER_NACOS_ACCESSKEY:""}
    secretKey: ${SW_CLUSTER_NACOS_SECRETKEY:""}
    internalComHost: ${SW_CLUSTER_INTERNAL_COM_HOST:""}
    internalComPort: ${SW_CLUSTER_INTERNAL_COM_PORT:-1}
core:
  selector: ${SW_CORE:default}
  default:
    # Mixed: 接收 Agent 数据，进行 L1 聚合和 L2 聚合
    # Receiver: 仅接收 Agent 数据，进行 L1 聚合
    # Aggregator: 仅进行 L2 聚合
    role: ${SW_CORE_ROLE:Mixed} # Mixed/Receiver/Aggregator
    restHost: ${SW_CORE_REST_HOST:0.0.0.0}
    restPort: ${SW_CORE_REST_PORT:12800}
    restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
    restMaxThreads: ${SW_CORE_REST_MAX_THREADS:200}
    restIdleTimeOut: ${SW_CORE_REST_IDLE_TIMEOUT:30000}
    restAcceptQueueSize: ${SW_CORE_REST_QUEUE_SIZE:0}
    httpMaxRequestHeaderSize: ${SW_CORE_HTTP_MAX_REQUEST_HEADER_SIZE:8192}
    gRPCHost: ${SW_CORE_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
    maxConcurrentCallsPerConnection: ${SW_CORE_GRPC_MAX_CONCURRENT_CALL:0}
    maxMessageSize: ${SW_CORE_GRPC_MAX_MESSAGE_SIZE:52428800}  # 50MB
    gRPCThreadPoolSize: ${SW_CORE_GRPC_THREAD_POOL_SIZE:-1}
    gRPCSslEnabled: ${SW_CORE_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_CORE_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_CORE_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAPath: ${SW_CORE_GRPC_SSL_TRUSTED_CA_PATH:""}
    downsampling:
      - Hour
      - Day
    # 是否启用数据清理执行器，关闭后自动删除指标数据也会关闭
    enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR:true}
    # 数据清理执行器运行周期，单位：分钟
    dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD:5}
    recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3}           # 记录数据保留天数（单位：天）
    metricsDataTTL: ${SW_CORE_METRICS_DATA_TTL:7}         # 指标数据保留天数（单位：天）
    # L1 聚合向 L2 聚合刷新的周期，单位：毫秒
    l1FlushPeriod: ${SW_CORE_L1_AGGREGATION_FLUSH_PERIOD:500}
    # 会话超时阈值，单位：毫秒，默认 70 秒
    storageSessionTimeout: ${SW_CORE_STORAGE_SESSION_TIMEOUT:70000}
    # 数据持久化周期，单位：秒，默认 25 秒
    persistentPeriod: ${SW_CORE_PERSISTENT_PERIOD:25}
    topNReportPeriod: ${SW_CORE_TOPN_REPORT_PERIOD:10}    # TopN 记录上报周期，单位：分钟
    # 激活代码中额外定义的模型列，这些列在聚合或查询中逻辑上非必须，
    # 会增加 OAP 的内存、网络和存储负载，但激活后可在 Kibana 等第三方工具中更容易查询。
    activeExtraModelColumns: ${SW_CORE_ACTIVE_EXTRA_MODEL_COLUMNS:false}
    # 服务名 + 实例名总长度最大值应小于 200
    serviceNameMaxLength: ${SW_SERVICE_NAME_MAX_LENGTH:70}
    # 服务缓存刷新周期，单位：秒，默认 10 秒
    serviceCacheRefreshInterval: ${SW_SERVICE_CACHE_REFRESH_INTERVAL:10}
    instanceNameMaxLength: ${SW_INSTANCE_NAME_MAX_LENGTH:70}
    # 服务名 + 端点名总长度最大值应小于 240
    endpointNameMaxLength: ${SW_ENDPOINT_NAME_MAX_LENGTH:150}
    # 可通过 GraphQL 搜索的 Span 标签键集合
    # key=value 最大长度小于 256，否则会被丢弃
    searchableTracesTags: ${SW_SEARCHABLE_TAG_KEYS:http.method,http.status_code,rpc.status_code,db.type,db.instance,mq.queue,mq.topic,mq.broker}
    # 可通过 GraphQL 搜索的日志标签键集合
    searchableLogsTags: ${SW_SEARCHABLE_LOGS_TAG_KEYS:level,http.status_code}
    # 可通过 GraphQL 搜索的告警标签键集合
    searchableAlarmTags: ${SW_SEARCHABLE_ALARM_TAG_KEYS:level}
    # 自动完成标签键查询的最大数量
    autocompleteTagKeysQueryMaxSize: ${SW_AUTOCOMPLETE_TAG_KEYS_QUERY_MAX_SIZE:100}
    # 自动完成标签值查询的最大数量
    autocompleteTagValuesQueryMaxSize: ${SW_AUTOCOMPLETE_TAG_VALUES_QUERY_MAX_SIZE:100}
    # 用于准备指标数据到存储的线程数
    prepareThreads: ${SW_CORE_PREPARE_THREADS:2}
    # 开启后根据 OpenAPI 定义自动对端点名进行分组
    enableEndpointNameGroupingByOpenapi: ${SW_CORE_ENABLE_ENDPOINT_NAME_GROUPING_BY_OPENAPI:true}
    # HTTP URI 模式识别的同步周期，单位：秒
    syncPeriodHttpUriRecognitionPattern: ${SW_CORE_SYNC_PERIOD_HTTP_URI_RECOGNITION_PATTERN:10}
    # HTTP URI 模式识别的训练周期，单位：秒
    trainingPeriodHttpUriRecognitionPattern: ${SW_CORE_TRAINING_PERIOD_HTTP_URI_RECOGNITION_PATTERN:60}
    # 每个服务用于进一步 URI 模式识别的最大 HTTP URI 数量
    maxHttpUrisNumberPerService: ${SW_CORE_MAX_HTTP_URIS_NUMBER_PER_SVR:3000}
    # 如果禁用层级关系，服务和实例的层级关系将不会构建，层级查询将返回空结果
    # 所有层级关系定义在 hierarchy-definition.yml 中
    # 注意：部分配置仅适用于 Kubernetes 环境
    enableHierarchy: ${SW_CORE_ENABLE_HIERARCHY:true}
    # 最大堆内存使用百分比（整数），默认 96%
    maxHeapMemoryUsagePercent: ${SW_CORE_MAX_HEAP_MEMORY_USAGE_PERCENT:96}
    # 最大直接内存使用量（字节），-1 表示无限制
    maxDirectMemoryUsage: ${SW_CORE_MAX_DIRECT_MEMORY_USAGE:-1}

storage:
  selector: ${SW_STORAGE:banyandb}
  banyandb:
    # 自 10.2.0 起，BanyanDB 配置已独立分离到 bydb.yaml 文件中
  elasticsearch:
    namespace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    connectTimeout: ${SW_STORAGE_ES_CONNECT_TIMEOUT:3000}
    socketTimeout: ${SW_STORAGE_ES_SOCKET_TIMEOUT:30000}
    responseTimeout: ${SW_STORAGE_ES_RESPONSE_TIMEOUT:15000}
    numHttpClientThread: ${SW_STORAGE_ES_NUM_HTTP_CLIENT_THREAD:0}
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    trustStorePath: ${SW_STORAGE_ES_SSL_JKS_PATH:""}
    trustStorePass: ${SW_STORAGE_ES_SSL_JKS_PASS:""}
    secretsManagementFile: ${SW_ES_SECRETS_MANAGEMENT_FILE:""} # 由第三方工具管理的包含用户名、密码等的密钥文件
    dayStep: ${SW_STORAGE_DAY_STEP:1}                         # 分钟/小时/天索引中代表的天数
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1} # 新索引的分片数
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:1} # 新索引的副本数
    # 为每个索引单独指定设置，优先级最高，会覆盖通用设置
    specificIndexSettings: ${SW_STORAGE_ES_SPECIFIC_INDEX_SETTINGS:""}
    # 超大数据集（如 trace segments）已由代码定义，以下配置可提升 ES 性能
    superDatasetDayStep: ${SW_STORAGE_ES_SUPER_DATASET_DAY_STEP:-1} # 超大数据集记录索引的天数步长，<0 时与 dayStep 相同
    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5} # 超大数据集分片因子：分片数 = indexShardsNumber × 此因子
    superDatasetIndexReplicasNumber: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_REPLICAS_NUMBER:0} # 超大数据集索引的副本数，默认 0
    indexTemplateOrder: ${SW_STORAGE_ES_INDEX_TEMPLATE_ORDER:0} # 索引模板的顺序
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:5000}           # 每达到此数量的请求执行一次异步批量写入
    batchOfBytes: ${SW_STORAGE_ES_BATCH_OF_BYTES:10485760}    # 控制 ES Bulk 刷新的最大 body 大小（字节）
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:5}          # 无论请求数量多少，每隔几秒强制 flush（秒）
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # 并发 Bulk 请求数
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:10000}
    scrollingBatchSize: ${SW_STORAGE_ES_SCROLLING_BATCH_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
    profileTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_PROFILE_TASK_SIZE:200}
    asyncProfilerTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_ASYNC_PROFILER_TASK_SIZE:200}
    profileDataQueryBatchSize: ${SW_STORAGE_ES_QUERY_PROFILE_DATA_BATCH_SIZE:100}
    oapAnalyzer: ${SW_STORAGE_ES_OAP_ANALYZER:"{\"analyzer\":{\"oap_analyzer\":{\"type\":\"stop\"}}}"} # OAP 分析器
    oapLogAnalyzer: ${SW_STORAGE_ES_OAP_LOG_ANALYZER:"{\"analyzer\":{\"oap_log_analyzer\":{\"type\":\"standard\"}}}"} # OAP 日志分析器，可自定义支持中文/日文等日志格式
    advanced: ${SW_STORAGE_ES_ADVANCED:""}
    # 将指标和记录索引分片到多个物理索引，每个指标/聚合函数一个索引模板
    logicSharding: ${SW_STORAGE_ES_LOGIC_SHARDING:false}
    # 启用自定义路由，可减少搜索的扇出范围，只查询匹配路由值的分片
    enableCustomRouting: ${SW_STORAGE_ES_ENABLE_CUSTOM_ROUTING:false}
  mysql:
    properties:
      jdbcUrl: ${SW_JDBC_URL:"jdbc:mysql://localhost:3306/swtest?rewriteBatchedStatements=true&allowMultiQueries=true"}
      dataSource.user: ${SW_DATA_SOURCE_USER:root}
      dataSource.password: ${SW_DATA_SOURCE_PASSWORD:root@1234}
      dataSource.cachePrepStmts: ${SW_DATA_SOURCE_CACHE_PREP_STMTS:true}
      dataSource.prepStmtCacheSize: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_SIZE:250}
      dataSource.prepStmtCacheSqlLimit: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_LIMIT:2048}
      dataSource.useServerPrepStmts: ${SW_DATA_SOURCE_USE_SERVER_PREP_STMTS:true}
    metadataQueryMaxSize: ${SW_STORAGE_MYSQL_QUERY_MAX_SIZE:5000}
    maxSizeOfBatchSql: ${SW_STORAGE_MAX_SIZE_OF_BATCH_SQL:2000}
    asyncBatchPersistentPoolSize: ${SW_STORAGE_ASYNC_BATCH_PERSISTENT_POOL_SIZE:4}
  postgresql:
    properties:
      jdbcUrl: ${SW_JDBC_URL:"jdbc:postgresql://localhost:5432/skywalking"}
      dataSource.user: ${SW_DATA_SOURCE_USER:postgres}
      dataSource.password: ${SW_DATA_SOURCE_PASSWORD:123456}
      dataSource.cachePrepStmts: ${SW_DATA_SOURCE_CACHE_PREP_STMTS:true}
      dataSource.prepStmtCacheSize: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_SIZE:250}
      dataSource.prepStmtCacheSqlLimit: ${SW_DATA_SOURCE_PREP_STMT_CACHE_SQL_LIMIT:2048}
      dataSource.useServerPrepStmts: ${SW_DATA_SOURCE_USE_SERVER_PREP_STMTS:true}
    metadataQueryMaxSize: ${SW_STORAGE_MYSQL_QUERY_MAX_SIZE:5000}
    maxSizeOfBatchSql: ${SW_STORAGE_MAX_SIZE_OF_BATCH_SQL:2000}
    asyncBatchPersistentPoolSize: ${SW_STORAGE_ASYNC_BATCH_PERSISTENT_POOL_SIZE:4}

agent-analyzer:
  selector: ${SW_AGENT_ANALYZER:default}
  default:
    # 默认采样率和默认追踪延迟时间，由 traceSamplingPolicySettingsFile 文件配置
    traceSamplingPolicySettingsFile: ${SW_TRACE_SAMPLING_POLICY_SETTINGS_FILE:trace-sampling-policy-settings.yml}
    slowDBAccessThreshold: ${SW_SLOW_DB_THRESHOLD:default:200,mongodb:100} # 慢数据库访问阈值，单位 ms
    forceSampleErrorSegment: ${SW_FORCE_SAMPLE_ERROR_SEGMENT:true} # 当采样机制启用时，强制保存某些错误 Segment（默认 true）
    segmentStatusAnalysisStrategy: ${SW_SEGMENT_STATUS_ANALYSIS_STRATEGY:FROM_SPAN_STATUS} # 从 Span 状态决定最终 Segment 状态，可选：FROM_SPAN_STATUS、FROM_ENTRY_SPAN、FROM_FIRST_SPAN
    # Nginx 和 Envoy 等 Agent 无法获取真实远程地址
    # 退出 Span 中 component 在此列表中的，不会生成客户端侧实例关系指标
    noUpstreamRealAddressAgents: ${SW_NO_UPSTREAM_REAL_ADDRESS:6000,9000}
    meterAnalyzerActiveFiles: ${SW_METER_ANALYZER_ACTIVE_FILES:datasource,threadpool,satellite,go-runtime,python-runtime,continuous-profiling,java-agent,go-agent} # 可进行 Meter 分析的文件列表，用逗号分隔
    slowCacheReadThreshold: ${SW_SLOW_CACHE_SLOW_READ_THRESHOLD:default:20,redis:10} # 慢缓存读取操作阈值，单位 ms
    slowCacheWriteThreshold: ${SW_SLOW_CACHE_SLOW_WRITE_THRESHOLD:default:20,redis:10} # 慢缓存写入操作阈值，单位 ms

log-analyzer:
  selector: ${SW_LOG_ANALYZER:default}
  default:
    lalFiles: ${SW_LOG_LAL_FILES:envoy-als,mesh-dp,mysql-slowsql,pgsql-slowsql,redis-slowsql,k8s-service,nginx,default}
    malFiles: ${SW_LOG_MAL_FILES:"nginx"}

event-analyzer:
  selector: ${SW_EVENT_ANALYZER:default}
  default:

receiver-sharing-server:
  selector: ${SW_RECEIVER_SHARING_SERVER:default}
  default:
    # HTTP 服务
    restHost: ${SW_RECEIVER_SHARING_REST_HOST:0.0.0.0}
    restPort: ${SW_RECEIVER_SHARING_REST_PORT:0}
    restContextPath: ${SW_RECEIVER_SHARING_REST_CONTEXT_PATH:/}
    restMaxThreads: ${SW_RECEIVER_SHARING_REST_MAX_THREADS:200}
    restIdleTimeOut: ${SW_RECEIVER_SHARING_REST_IDLE_TIMEOUT:30000}
    restAcceptQueueSize: ${SW_RECEIVER_SHARING_REST_QUEUE_SIZE:0}
    httpMaxRequestHeaderSize: ${SW_RECEIVER_SHARING_HTTP_MAX_REQUEST_HEADER_SIZE:8192}
    # gRPC 服务
    gRPCHost: ${SW_RECEIVER_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_RECEIVER_GRPC_PORT:0}
    maxConcurrentCallsPerConnection: ${SW_RECEIVER_GRPC_MAX_CONCURRENT_CALL:0}
    maxMessageSize: ${SW_RECEIVER_GRPC_MAX_MESSAGE_SIZE:52428800} # 50MB
    gRPCThreadPoolSize: ${SW_RECEIVER_GRPC_THREAD_POOL_SIZE:0}
    gRPCSslEnabled: ${SW_RECEIVER_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_RECEIVER_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_RECEIVER_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAsPath: ${SW_RECEIVER_GRPC_SSL_TRUSTED_CAS_PATH:""}
    authentication: ${SW_AUTHENTICATION:""}
receiver-register:
  selector: ${SW_RECEIVER_REGISTER:default}
  default:

receiver-trace:
  selector: ${SW_RECEIVER_TRACE:default}
  default:

receiver-jvm:
  selector: ${SW_RECEIVER_JVM:default}
  default:

receiver-clr:
  selector: ${SW_RECEIVER_CLR:default}
  default:

receiver-profile:
  selector: ${SW_RECEIVER_PROFILE:default}
  default:

receiver-async-profiler:
  selector: ${SW_RECEIVER_ASYNC_PROFILER:default}
  default:
    # 用于管理可接收的 JFR 文件最大大小，单位 Byte，默认 30MB
    jfrMaxSize: ${SW_RECEIVER_ASYNC_PROFILER_JFR_MAX_SIZE:31457280}
    # 是否使用内存文件模式解析 JFR（默认 true），内存模式对文件系统限制少，但耗内存更多；
    # 物理文件模式解析时内存占用少，对大文件更友好，但容器 tmp 目录空间不足可能导致崩溃
    memoryParserEnabled: ${SW_RECEIVER_ASYNC_PROFILER_MEMORY_PARSER_ENABLED:true}

receiver-zabbix:
  selector: ${SW_RECEIVER_ZABBIX:-}
  default:
    port: ${SW_RECEIVER_ZABBIX_PORT:10051}
    host: ${SW_RECEIVER_ZABBIX_HOST:0.0.0.0}
    activeFiles: ${SW_RECEIVER_ZABBIX_ACTIVE_FILES:agent}

service-mesh:
  selector: ${SW_SERVICE_MESH:default}
  default:

envoy-metric:
  selector: ${SW_ENVOY_METRIC:default}
  default:
    acceptMetricsService: ${SW_ENVOY_METRIC_SERVICE:true}
    alsHTTPAnalysis: ${SW_ENVOY_METRIC_ALS_HTTP_ANALYSIS:""}
    alsTCPAnalysis: ${SW_ENVOY_METRIC_ALS_TCP_ANALYSIS:""}
    # k8sServiceNameRule 允许通过 Kubernetes 元数据自定义服务名，可用变量如 pod、service
    # 示例：${service.metadata.name}-${pod.metadata.labels.version}
    k8sServiceNameRule: ${K8S_SERVICE_NAME_RULE:"${pod.metadata.labels.(service.istio.io/canonical-name)}.${pod.metadata.namespace}"}
    istioServiceNameRule: ${ISTIO_SERVICE_NAME_RULE:"${serviceEntry.metadata.name}.${serviceEntry.metadata.namespace}"}
    # 从 Istio ServiceEntries 查找服务信息时，排除某些命名空间的 ServiceEntries
    istioServiceEntryIgnoredNamespaces: ${SW_ISTIO_SERVICE_ENTRY_IGNORED_NAMESPACES:""}
    gRPCHost: ${SW_ALS_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_ALS_GRPC_PORT:0}
    maxConcurrentCallsPerConnection: ${SW_ALS_GRPC_MAX_CONCURRENT_CALL:0}
    maxMessageSize: ${SW_ALS_GRPC_MAX_MESSAGE_SIZE:0}
    gRPCThreadPoolSize: ${SW_ALS_GRPC_THREAD_POOL_SIZE:0}
    gRPCSslEnabled: ${SW_ALS_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_ALS_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_ALS_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAsPath: ${SW_ALS_GRPC_SSL_TRUSTED_CAS_PATH:""}

kafka-fetcher:
  selector: ${SW_KAFKA_FETCHER:-}
  default:
    bootstrapServers: ${SW_KAFKA_FETCHER_SERVERS:localhost:9092}
    namespace: ${SW_NAMESPACE:""}
    partitions: ${SW_KAFKA_FETCHER_PARTITIONS:3}
    replicationFactor: ${SW_KAFKA_FETCHER_PARTITIONS_FACTOR:2}
    enableNativeProtoLog: ${SW_KAFKA_FETCHER_ENABLE_NATIVE_PROTO_LOG:true}
    enableNativeJsonLog: ${SW_KAFKA_FETCHER_ENABLE_NATIVE_JSON_LOG:true}
    consumers: ${SW_KAFKA_FETCHER_CONSUMERS:1}
    kafkaHandlerThreadPoolSize: ${SW_KAFKA_HANDLER_THREAD_POOL_SIZE:-1}
    kafkaHandlerThreadPoolQueueSize: ${SW_KAFKA_HANDLER_THREAD_POOL_QUEUE_SIZE:-1}

cilium-fetcher:
  selector: ${SW_CILIUM_FETCHER:-}
  default:
    peerHost: ${SW_CILIUM_FETCHER_PEER_HOST:hubble-peer.kube-system.svc.cluster.local}
    peerPort: ${SW_CILIUM_FETCHER_PEER_PORT:80}
    fetchFailureRetrySecond: ${SW_CILIUM_FETCHER_FETCH_FAILURE_RETRY_SECOND:10}
    sslConnection: ${SW_CILIUM_FETCHER_SSL_CONNECTION:false}
    sslPrivateKeyFile: ${SW_CILIUM_FETCHER_PRIVATE_KEY_FILE_PATH:}
    sslCertChainFile: ${SW_CILIUM_FETCHER_CERT_CHAIN_FILE_PATH:}
    sslCaFile: ${SW_CILIUM_FETCHER_CA_FILE_PATH:}
    convertClientAsServerTraffic: ${SW_CILIUM_FETCHER_CONVERT_CLIENT_AS_SERVER_TRAFFIC:true}

receiver-meter:
  selector: ${SW_RECEIVER_METER:default}
  default:

receiver-otel:
  selector: ${SW_OTEL_RECEIVER:default}
  default:
    enabledHandlers: ${SW_OTEL_RECEIVER_ENABLED_HANDLERS:"otlp-metrics,otlp-logs"}
    enabledOtelMetricsRules: ${SW_OTEL_RECEIVER_ENABLED_OTEL_METRICS_RULES:"apisix,nginx/*,k8s/*,istio-controlplane,vm,mysql/*,postgresql/*,oap,aws-eks/*,windows,aws-s3/*,aws-dynamodb/*,aws-gateway/*,redis/*,elasticsearch/*,rabbitmq/*,mongodb/*,kafka/*,pulsar/*,bookkeeper/*,rocketmq/*,clickhouse/*,activemq/*,kong/*"}

receiver-zipkin:
  selector: ${SW_RECEIVER_ZIPKIN:-}
  default:
    # 可搜索的 Span 标签键集合，key=value 最大长度小于 256，否则丢弃
    searchableTracesTags: ${SW_ZIPKIN_SEARCHABLE_TAG_KEYS:http.method}
    # 采样率精度 1/10000，范围 0~10000
    sampleRate: ${SW_ZIPKIN_SAMPLE_RATE:10000}
    # 以下配置用于从 HTTP 收集 Zipkin Trace
    enableHttpCollector: ${SW_ZIPKIN_HTTP_COLLECTOR_ENABLED:true}
    restHost: ${SW_RECEIVER_ZIPKIN_REST_HOST:0.0.0.0}
    restPort: ${SW_RECEIVER_ZIPKIN_REST_PORT:9411}
    restContextPath: ${SW_RECEIVER_ZIPKIN_REST_CONTEXT_PATH:/}
    restMaxThreads: ${SW_RECEIVER_ZIPKIN_REST_MAX_THREADS:200}
    restIdleTimeOut: ${SW_RECEIVER_ZIPKIN_REST_IDLE_TIMEOUT:30000}
    restAcceptQueueSize: ${SW_RECEIVER_ZIPKIN_REST_QUEUE_SIZE:0}
    # 以下配置用于从 Kafka 收集 Zipkin Trace
    enableKafkaCollector: ${SW_ZIPKIN_KAFKA_COLLECTOR_ENABLED:false}
    kafkaBootstrapServers: ${SW_ZIPKIN_KAFKA_SERVERS:localhost:9092}
    kafkaGroupId: ${SW_ZIPKIN_KAFKA_GROUP_ID:zipkin}
    kafkaTopic: ${SW_ZIPKIN_KAFKA_TOPIC:zipkin}
    # Kafka 消费者配置，JSON 格式的 Properties，若与上方键重复则覆盖
    kafkaConsumerConfig: ${SW_ZIPKIN_KAFKA_CONSUMER_CONFIG:"{\"auto.offset.reset\":\"earliest\",\"enable.auto.commit\":true}"}
    # Topic 消费者数量
    kafkaConsumers: ${SW_ZIPKIN_KAFKA_CONSUMERS:1}
    kafkaHandlerThreadPoolSize: ${SW_ZIPKIN_KAFKA_HANDLER_THREAD_POOL_SIZE:-1}
    kafkaHandlerThreadPoolQueueSize: ${SW_ZIPKIN_KAFKA_HANDLER_THREAD_POOL_QUEUE_SIZE:-1}

receiver-browser:
  selector: ${SW_RECEIVER_BROWSER:default}
  default:
    # 采样率精度 1/10000，10000 表示默认 100% 采样
    sampleRate: ${SW_RECEIVER_BROWSER_SAMPLE_RATE:10000}

receiver-log:
  selector: ${SW_RECEIVER_LOG:default}
  default:

query:
  selector: ${SW_QUERY:graphql}
  graphql:
    # 启用日志测试 API 来测试 LAL（注意：此 API 会评估不受信任的代码，具有风险，仅在完全信任用户时启用）
    enableLogTestTool: ${SW_QUERY_GRAPHQL_ENABLE_LOG_TEST_TOOL:false}
    # GraphQL 查询允许的最大复杂度，超过阈值将被中止
    maxQueryComplexity: ${SW_QUERY_MAX_QUERY_COMPLEXITY:3000}
    # 允许用户添加、禁用和更新 UI 模板
    enableUpdateUITemplate: ${SW_ENABLE_UPDATE_UI_TEMPLATE:false}
    # “On demand log” 允许实时获取 Pod 容器日志（可能暴露日志中的秘密，需要为 OAP cluster role 添加权限）
    enableOnDemandPodLog: ${SW_ENABLE_ON_DEMAND_POD_LOG:false}

# 此模块用于 Zipkin 查询 API 并支持 zipkin-lens UI
query-zipkin:
  selector: ${SW_QUERY_ZIPKIN:-}
  default:
    # HTTP 服务
    restHost: ${SW_QUERY_ZIPKIN_REST_HOST:0.0.0.0}
    restPort: ${SW_QUERY_ZIPKIN_REST_PORT:9412}
    restContextPath: ${SW_QUERY_ZIPKIN_REST_CONTEXT_PATH:/zipkin}
    restMaxThreads: ${SW_QUERY_ZIPKIN_REST_MAX_THREADS:200}
    restIdleTimeOut: ${SW_QUERY_ZIPKIN_REST_IDLE_TIMEOUT:30000}
    restAcceptQueueSize: ${SW_QUERY_ZIPKIN_REST_QUEUE_SIZE:0}
    # 默认 traces 查询回溯时间，1 天（毫秒）
    lookback: ${SW_QUERY_ZIPKIN_LOOKBACK:86400000}
    # 服务名、远程服务名、Span 名的 Cache-Control max-age（秒）
    namesMaxAge: ${SW_QUERY_ZIPKIN_NAMES_MAX_AGE:300}
    # UI 查询 traces 的默认最大数量
    uiQueryLimit: ${SW_QUERY_ZIPKIN_UI_QUERY_LIMIT:10}
    # UI 搜索 traces 的默认回溯时间，15 分钟（毫秒）
    uiDefaultLookback: ${SW_QUERY_ZIPKIN_UI_DEFAULT_LOOKBACK:900000}

# 此模块用于 PromQL API
promql:
  selector: ${SW_PROMQL:default}
  default:
    # HTTP 服务
    restHost: ${SW_PROMQL_REST_HOST:0.0.0.0}
    restPort: ${SW_PROMQL_REST_PORT:9090}
    restContextPath: ${SW_PROMQL_REST_CONTEXT_PATH:/}
    restMaxThreads: ${SW_PROMQL_REST_MAX_THREADS:200}
    restIdleTimeOut: ${SW_PROMQL_REST_IDLE_TIMEOUT:30000}
    restAcceptQueueSize: ${SW_PROMQL_REST_QUEUE_SIZE:0}
    # 用于 API buildInfo，设置模拟的构建信息
    buildInfoVersion: ${SW_PROMQL_BUILD_INFO_VERSION:"2.45.0"}
    buildInfoRevision: ${SW_PROMQL_BUILD_INFO_REVISION:""}
    buildInfoBranch: ${SW_PROMQL_BUILD_INFO_BRANCH:""}
    buildInfoBuildUser: ${SW_PROMQL_BUILD_INFO_BUILD_USER:""}
    buildInfoBuildDate: ${SW_PROMQL_BUILD_INFO_BUILD_DATE:""}
    buildInfoGoVersion: ${SW_PROMQL_BUILD_INFO_GO_VERSION:""}

# 此模块用于 LogQL API
logql:
  selector: ${SW_LOGQL:default}
  default:
    # HTTP 服务
    restHost: ${SW_LOGQL_REST_HOST:0.0.0.0}
    restPort: ${SW_LOGQL_REST_PORT:3100}
    restContextPath: ${SW_LOGQL_REST_CONTEXT_PATH:/}
    restMaxThreads: ${SW_LOGQL_REST_MAX_THREADS:200}
    restIdleTimeOut: ${SW_LOGQL_REST_IDLE_TIMEOUT:30000}
    restAcceptQueueSize: ${SW_LOGQL_REST_QUEUE_SIZE:0}

alarm:
  selector: ${SW_ALARM:default}
  default:

telemetry:
  selector: ${SW_TELEMETRY:prometheus}
  none:
  prometheus:
    host: ${SW_TELEMETRY_PROMETHEUS_HOST:0.0.0.0}
    port: ${SW_TELEMETRY_PROMETHEUS_PORT:1234}
    sslEnabled: ${SW_TELEMETRY_PROMETHEUS_SSL_ENABLED:false}
    sslKeyPath: ${SW_TELEMETRY_PROMETHEUS_SSL_KEY_PATH:""}
    sslCertChainPath: ${SW_TELEMETRY_PROMETHEUS_SSL_CERT_CHAIN_PATH:""}

configuration:
  selector: ${SW_CONFIGURATION:none}
  none:
  grpc:
    host: ${SW_DCS_SERVER_HOST:""}
    port: ${SW_DCS_SERVER_PORT:80}
    clusterName: ${SW_DCS_CLUSTER_NAME:SkyWalking}
    period: ${SW_DCS_PERIOD:20}
    maxInboundMessageSize: ${SW_DCS_MAX_INBOUND_MESSAGE_SIZE:4194304}
  apollo:
    apolloMeta: ${SW_CONFIG_APOLLO:http://localhost:8080}
    apolloCluster: ${SW_CONFIG_APOLLO_CLUSTER:default}
    apolloEnv: ${SW_CONFIG_APOLLO_ENV:""}
    appId: ${SW_CONFIG_APOLLO_APP_ID:skywalking}
  zookeeper:
    period: ${SW_CONFIG_ZK_PERIOD:60} # 同步周期，单位秒，默认每 60 秒拉取一次
    namespace: ${SW_CONFIG_ZK_NAMESPACE:/default}
    hostPort: ${SW_CONFIG_ZK_HOST_PORT:localhost:2181}
    # 重试策略
    baseSleepTimeMs: ${SW_CONFIG_ZK_BASE_SLEEP_TIME_MS:1000} # 重试之间初始等待时间（毫秒）
    maxRetries: ${SW_CONFIG_ZK_MAX_RETRIES:3}               # 最大重试次数
  etcd:
    period: ${SW_CONFIG_ETCD_PERIOD:60} # 同步周期，单位秒，默认每 60 秒拉取一次
    endpoints: ${SW_CONFIG_ETCD_ENDPOINTS:http://localhost:2379}
    namespace: ${SW_CONFIG_ETCD_NAMESPACE:/skywalking}
    authentication: ${SW_CONFIG_ETCD_AUTHENTICATION:false}
    user: ${SW_CONFIG_ETCD_USER:}
    password: ${SW_CONFIG_ETCD_PASSWORD:}
  consul:
    # Consul 主机和端口，逗号分隔，例如 1.2.3.4:8500
    hostAndPorts: ${SW_CONFIG_CONSUL_HOST_AND_PORTS:1.2.3.4:8500}
    # 同步周期，单位秒，默认 60 秒
    period: ${SW_CONFIG_CONSUL_PERIOD:60}
    # Consul aclToken
    aclToken: ${SW_CONFIG_CONSUL_ACL_TOKEN:""}
  k8s-configmap:
    period: ${SW_CONFIG_CONFIGMAP_PERIOD:60}
    namespace: ${SW_CLUSTER_K8S_NAMESPACE:default}
    labelSelector: ${SW_CLUSTER_K8S_LABEL:app=collector,release=skywalking}
  nacos:
    # Nacos 服务地址
    serverAddr: ${SW_CONFIG_NACOS_SERVER_ADDR:127.0.0.1}
    # Nacos 服务端口
    port: ${SW_CONFIG_NACOS_SERVER_PORT:8848}
    # Nacos 配置分组
    group: ${SW_CONFIG_NACOS_SERVER_GROUP:skywalking}
    # Nacos 配置命名空间
    namespace: ${SW_CONFIG_NACOS_SERVER_NAMESPACE:}
    # 同步周期，单位秒，默认每 60 秒拉取一次
    period: ${SW_CONFIG_NACOS_PERIOD:60}
    # Nacos 认证用户名
    username: ${SW_CONFIG_NACOS_USERNAME:""}
    password: ${SW_CONFIG_NACOS_PASSWORD:""}
    # Nacos 认证 accessKey
    accessKey: ${SW_CONFIG_NACOS_ACCESSKEY:""}
    secretKey: ${SW_CONFIG_NACOS_SECRETKEY:""}

exporter:
  selector: ${SW_EXPORTER:-}
  default:
    # gRPC 导出器
    enableGRPCMetrics: ${SW_EXPORTER_ENABLE_GRPC_METRICS:false}
    gRPCTargetHost: ${SW_EXPORTER_GRPC_HOST:127.0.0.1}
    gRPCTargetPort: ${SW_EXPORTER_GRPC_PORT:9870}
    # Kafka 导出器
    enableKafkaTrace: ${SW_EXPORTER_ENABLE_KAFKA_TRACE:false}
    enableKafkaLog: ${SW_EXPORTER_ENABLE_KAFKA_LOG:false}
    kafkaBootstrapServers: ${SW_EXPORTER_KAFKA_SERVERS:localhost:9092}
    # Kafka 生产者配置，JSON 格式的 Properties
    kafkaProducerConfig: ${SW_EXPORTER_KAFKA_PRODUCER_CONFIG:""}
    kafkaTopicTrace: ${SW_EXPORTER_KAFKA_TOPIC_TRACE:skywalking-export-trace}
    kafkaTopicLog: ${SW_EXPORTER_KAFKA_TOPIC_LOG:skywalking-export-log}
    exportErrorStatusTraceOnly: ${SW_EXPORTER_KAFKA_TRACE_FILTER_ERROR:false}

health-checker:
  selector: ${SW_HEALTH_CHECKER:-}
  default:
    checkIntervalSeconds: ${SW_HEALTH_CHECKER_INTERVAL_SECONDS:5}

status-query:
  selector: ${SW_STATUS_QUERY:default}
  default:
    # 过滤配置中包含秘密的关键字列表，用逗号分隔
    keywords4MaskingSecretsOfConfig: ${SW_DEBUGGING_QUERY_KEYWORDS_FOR_MASKING_SECRETS:user,password,token,accessKey,secretKey,authentication}

configuration-discovery:
  selector: ${SW_CONFIGURATION_DISCOVERY:default}
  default:
    disableMessageDigest: ${SW_DISABLE_MESSAGE_DIGEST:false}

receiver-event:
  selector: ${SW_RECEIVER_EVENT:default}
  default:

receiver-ebpf:
  selector: ${SW_RECEIVER_EBPF:default}
  default:
    # 持续剖析策略缓存时间，单位秒
    continuousPolicyCacheTimeout: ${SW_CONTINUOUS_POLICY_CACHE_TIMEOUT:60}
    gRPCHost: ${SW_EBPF_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_EBPF_GRPC_PORT:0}
    maxConcurrentCallsPerConnection: ${SW_EBPF_GRPC_MAX_CONCURRENT_CALL:0}
    maxMessageSize: ${SW_EBPF_ALS_GRPC_MAX_MESSAGE_SIZE:0}
    gRPCThreadPoolSize: ${SW_EBPF_GRPC_THREAD_POOL_SIZE:0}
    gRPCSslEnabled: ${SW_EBPF_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_EBPF_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_EBPF_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAsPath: ${SW_EBPF_GRPC_SSL_TRUSTED_CAS_PATH:""}

receiver-telegraf:
  selector: ${SW_RECEIVER_TELEGRAF:default}
  default:
    activeFiles: ${SW_RECEIVER_TELEGRAF_ACTIVE_FILES:vm}

aws-firehose:
  selector: ${SW_RECEIVER_AWS_FIREHOSE:default}
  default:
    host: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_HOST:0.0.0.0}
    port: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_PORT:12801}
    contextPath: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_CONTEXT_PATH:/}
    maxThreads: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_MAX_THREADS:200}
    idleTimeOut: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_IDLE_TIME_OUT:30000}
    acceptQueueSize: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_ACCEPT_QUEUE_SIZE:0}
    maxRequestHeaderSize: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_MAX_REQUEST_HEADER_SIZE:8192}
    firehoseAccessKey: ${SW_RECEIVER_AWS_FIREHOSE_ACCESS_KEY:}
    enableTLS: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_ENABLE_TLS:false}
    tlsKeyPath: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_TLS_KEY_PATH:}
    tlsCertChainPath: ${SW_RECEIVER_AWS_FIREHOSE_HTTP_TLS_CERT_CHAIN_PATH:}

ai-pipeline:
  selector: ${SW_AI_PIPELINE:default}
  default:
    uriRecognitionServerAddr: ${SW_AI_PIPELINE_URI_RECOGNITION_SERVER_ADDR:}
    uriRecognitionServerPort: ${SW_AI_PIPELINE_URI_RECOGNITION_SERVER_PORT:17128}
    baselineServerAddr: ${SW_API_PIPELINE_BASELINE_SERVICE_HOST:}
    baselineServerPort: ${SW_API_PIPELINE_BASELINE_SERVICE_PORT:18080}
```

#### 1.2.3 启动配置

`docker-compose.yml`

```yaml
services:
  oap:
    image: registry.cn-beijing.aliyuncs.com/witter/skywalking-oap-server:10.2.0
    container_name: oap-server
    restart: unless-stopped
    ports:
      - "11800:11800"   # gRPC 上报端口（Agent/Satellite 连这里）
      - "12800:12800"   # REST/UI 端口
    volumes:
      - ./oap/config/application.yml:/skywalking/config/application.yml
    environment:
      - SW_STORAGE=elasticsearch
      - SW_STORAGE_ES_CLUSTER_NODES=192.168.8.99:9200
      # 调优参数（Span/Trace量大时）
      - SW_STORAGE_ES_BULK_ACTIONS=5000
      - SW_STORAGE_ES_BATCH_OF_BYTES=10485760
      - SW_STORAGE_ES_FLUSH_INTERVAL=3
      - SW_STORAGE_ES_CONCURRENT_REQUESTS=3
      - SW_CORE_PERSISTENT_PERIOD=15
      - SW_ES_USER=elastic
      - SW_ES_PASSWORD=elastic
      - SW_STORAGE_ES_HTTP_PROTOCOL=http
      # 可选：命名空间（多租户场景）
      # - SW_NAMESPACE=
      - SW_TELEMETRY=prometheus
      - JAVA_OPTS=-Xms2048m -Xmx2048m
    networks:
      - skywalking

  ui:
    image: registry.cn-beijing.aliyuncs.com/witter/skywalking-ui:10.2.0
    container_name: ui
    restart: always
    depends_on:
      - oap
    ports:
      - "8081:8080"
    environment:
      - SW_OAP_ADDRESS=http://oap:12800
      - SW_ZIPKIN_ADDRESS=http://oap:9412
    networks:
      - skywalking

  satellite:
    image: registry.cn-beijing.aliyuncs.com/witter/skywalking-satellite:v1.3.0
    container_name: satellite
    restart: unless-stopped
    depends_on:
      - oap
    ports:
      - "11801:11801"
      - "11234:11234"
    environment:
      SATELLITE_GRPC_ADDRESS: ":11801"
      SATELLITE_LOGGER_LEVEL: info
      SATELLITE_TELEMETRY_EXPORT_TYPE: "none"
      SATELLITE_GRPC_CLIENT_FINDER: "static"
      SATELLITE_GRPC_CLIENT: "192.168.8.104:11800"
      SATELLITE_QUEUE_EVENT_BUFFER_SIZE: "50000"
      # 如果启用了 TLS / 认证，可以继续设置相关变量
      # SATELLITE_KAFKA_ENABLE_TLS: "true"
      # SATELLITE_KAFKA_CLIENT_PEM_PATH: "/etc/satellite/certs/client.pem"
    volumes:
      - ./satellite/satellite_config.yaml:/skywalking/config/satellite_config.yaml:ro
      #- ./certs:/etc/satellite/certs:ro
    networks:
      - skywalking

networks:
  skywalking:
    driver: bridge
```

#### 1.2.4 es写入索引名

`sw_*`



## 2.Agent配置

### 2.1 Java Agent

1. 下载&配置文档：https://skywalking.apache.org/docs/skywalking-java/v9.6.0/readme/
2. agent环境变量：https://skywalking.apache.org/docs/skywalking-java/v9.6.0/en/setup/service-agent/java-agent/configurations/
3. 插件名列表：https://skywalking.apache.org/docs/skywalking-java/v9.6.0/en/setup/service-agent/java-agent/plugin-list/

```shell
# 在java服务启动项里添加
-javaagent:/skywalking-agent/skywalking-agent.jar
-Dskywalking.agent.service_name=sk-biz-gateway
-Dskywalking.collector.backend_service=192.168.8.104:11800
-Dskywalking.agent.sample_n_per_3_secs=-1	# 采样策略，值设为 -1 表示全采样，不丢弃任何链路
-Dskywalking.agent.span_limit_per_segment=50000 # 单个 Segment 中 Span 的最大数量(默认值为300)

# 如有插件不兼容情况，添加下面的参数
-Dskywalking.plugin.exclude_plugins=tomcat-7.x/8.x,tomcat-10.x,tomcat-thread-pool
```




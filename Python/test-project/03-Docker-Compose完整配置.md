# Docker-Compose完整配置

## 文档信息

- **版本**: v1.0
- **日期**: 2026-04-12
- **硬件配置**: 4核CPU / 8GB内存 / 160GB磁盘

---

## 目录

1. [项目目录结构](#1-项目目录结构)
2. [Docker Compose主配置](#2-docker-compose主配置)
3. [服务配置详解](#3-服务配置详解)
4. [环境变量配置](#4-环境变量配置)
5. [配置文件](#5-配置文件)
6. [网络与存储](#6-网络与存储)
7. [启动与运维](#7-启动与运维)

---

## 1. 项目目录结构

```
/data/
├── docker-compose.yml          # 主配置文件
├── .env                        # 环境变量
├── docker-compose.override.yml # 本地覆盖配置(可选)
│
├── mysql/
│   ├── data/                   # MySQL数据目录
│   ├── init/                   # 初始化脚本
│   │   └── 01-init.sql
│   └── my.cnf                  # MySQL配置
│
├── redis/
│   ├── data/                   # Redis数据目录
│   └── redis.conf              # Redis配置(可选)
│
├── es/
│   ├── data/                   # ES数据目录
│   └── elasticsearch.yml       # ES配置
│
├── rabbitmq/
│   └── data/                   # RabbitMQ数据目录
│
├── prometheus/
│   ├── prometheus.yml          # Prometheus配置
│   ├── alert_rules.yml         # 告警规则
│   └── data/                   # 数据目录
│
├── grafana/
│   ├── data/                   # Grafana数据目录
│   └── provisioning/           # 自动配置
│       ├── dashboards/
│       └── datasources/
│
├── nginx/
│   ├── nginx.conf              # Nginx配置
│   ├── ssl/                    # SSL证书
│   └── html/                   # 前端静态文件
│
├── app/                        # FastAPI应用
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── config.py
│   ├── models/
│   ├── routers/
│   ├── services/
│   ├── tasks/
│   └── utils/
│
├── logs/                       # 日志目录
├── backups/                    # 备份目录
└── scripts/                    # 运维脚本
    ├── backup.sh
    ├── cleanup.sh
    └── monitor.sh
```

---

## 2. Docker Compose主配置

### 2.1 完整配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================
  # MySQL 8.0 - 主数据库
  # 内存限制: 1GB
  # ============================================
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ./mysql/data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d
      - ./mysql/my.cnf:/etc/mysql/conf.d/my.cnf:ro
      - ./backups:/backups
    ports:
      - "3306:3306"
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          memory: 512M
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ============================================
  # Redis 7 - 缓存与消息代理
  # 内存限制: 512MB
  # ============================================
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: >
      redis-server
      --appendonly yes
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --requirepass ${REDIS_PASSWORD}
    volumes:
      - ./redis/data:/data
    ports:
      - "6379:6379"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 128M
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================
  # Elasticsearch 8.x - 全文搜索
  # 内存限制: 2GB (1GB堆内存)
  # ============================================
  elasticsearch:
    image: elasticsearch:8.11.0
    container_name: elasticsearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - cluster.name=data-platform
      - node.name=es-single
      - bootstrap.memory_lock=false
      - indices.fielddata.cache.size=30%
    volumes:
      - ./es/data:/usr/share/elasticsearch/data
      - ./es/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
    ports:
      - "9200:9200"
      - "9300:9300"
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
        reservations:
          memory: 1G
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ============================================
  # RabbitMQ 3 - 消息队列
  # 内存限制: 512MB
  # ============================================
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      - ./rabbitmq/data:/var/lib/rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 128M
    networks:
      - backend
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "status"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ============================================
  # FastAPI应用 - 主服务
  # 内存限制: 1GB (4 workers)
  # ============================================
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: data-platform-app
    restart: unless-stopped
    environment:
      - APP_ENV=production
      - DEBUG=false
      - WORKERS=4
      - MAX_CONNECTIONS=100
      - PROCESS_BATCH_SIZE=500
      - MYSQL_HOST=mysql
      - MYSQL_PORT=3306
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - ES_HOST=elasticsearch
      - ES_PORT=9200
      - RABBITMQ_URL=amqp://${RABBITMQ_USER}:${RABBITMQ_PASSWORD}@rabbitmq:5672/
      - SECRET_KEY=${SECRET_KEY}
      - ALGORITHM=${ALGORITHM}
      - ACCESS_TOKEN_EXPIRE_MINUTES=${ACCESS_TOKEN_EXPIRE_MINUTES}
    volumes:
      - ./logs:/app/logs
      - ./backups:/app/backups
    ports:
      - "8000:8000"
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          memory: 256M
    networks:
      - backend
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
      rabbitmq:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ============================================
  # Celery Worker - 后台任务
  # 内存限制: 512MB (4并发)
  # ============================================
  celery-worker:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: celery-worker
    restart: unless-stopped
    command: celery -A tasks worker --loglevel=info --concurrency=4 --prefetch-multiplier=1 --max-tasks-per-child=50
    environment:
      - APP_ENV=production
      - DEBUG=false
      - MYSQL_HOST=mysql
      - MYSQL_PORT=3306
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - ES_HOST=elasticsearch
      - ES_PORT=9200
      - RABBITMQ_URL=amqp://${RABBITMQ_USER}:${RABBITMQ_PASSWORD}@rabbitmq:5672/
      - SECRET_KEY=${SECRET_KEY}
    volumes:
      - ./logs:/app/logs
      - ./backups:/app/backups
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 128M
    networks:
      - backend
    depends_on:
      - mysql
      - redis
      - rabbitmq

  # ============================================
  # Celery Beat - 定时任务调度
  # ============================================
  celery-beat:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: celery-beat
    restart: unless-stopped
    command: celery -A tasks beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    environment:
      - APP_ENV=production
      - MYSQL_HOST=mysql
      - MYSQL_PORT=3306
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - RABBITMQ_URL=amqp://${RABBITMQ_USER}:${RABBITMQ_PASSWORD}@rabbitmq:5672/
    volumes:
      - ./logs:/app/logs
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
    networks:
      - backend
    depends_on:
      - mysql
      - redis
      - rabbitmq

  # ============================================
  # Prometheus - 监控采集
  # 内存限制: 256MB
  # ============================================
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - ./prometheus/data:/prometheus
    ports:
      - "9090:9090"
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
        reservations:
          memory: 64M
    networks:
      - backend

  # ============================================
  # Grafana - 可视化
  # 内存限制: 256MB
  # ============================================
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://localhost:3000
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource
    volumes:
      - ./grafana/data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
        reservations:
          memory: 64M
    networks:
      - backend
    depends_on:
      - prometheus

  # ============================================
  # Nginx - 反向代理
  # 内存限制: 128MB
  # ============================================
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/html:/usr/share/nginx/html:ro
      - ./logs/nginx:/var/log/nginx
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 128M
        reservations:
          memory: 32M
    networks:
      - backend
    depends_on:
      - app

# ============================================
# 网络配置
# ============================================
networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

# ============================================
# 卷配置(可选，使用bind mount已在服务中配置)
# ============================================
volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local
  es_data:
    driver: local
  rabbitmq_data:
    driver: local
  prometheus_data:
    driver: local
  grafana_data:
    driver: local
```

---

## 3. 服务配置详解

### 3.1 资源分配汇总

| 服务 | CPU | 内存限制 | 内存预留 | 说明 |
|------|-----|----------|----------|------|
| MySQL | 1.0 | 1GB | 512MB | 数据库 |
| Redis | 0.5 | 512MB | 128MB | 缓存 |
| Elasticsearch | 1.0 | 2GB | 1GB | 搜索引擎 |
| RabbitMQ | 0.5 | 512MB | 128MB | 消息队列 |
| FastAPI | 1.0 | 1GB | 256MB | 主应用 |
| Celery Worker | 0.5 | 512MB | 128MB | 后台任务 |
| Celery Beat | 0.25 | 256MB | - | 定时调度 |
| Prometheus | 0.25 | 256MB | 64MB | 监控 |
| Grafana | 0.25 | 256MB | 64MB | 可视化 |
| Nginx | 0.25 | 128MB | 32MB | 代理 |
| **总计** | **~5.5** | **~6.5GB** | **~2.5GB** | **8GB系统** |

---

## 4. 环境变量配置

### 4.1 环境变量文件

```bash
# .env - 生产环境变量

# ============================================
# MySQL配置
# ============================================
MYSQL_ROOT_PASSWORD=Root@123456
MYSQL_DATABASE=data_platform
MYSQL_USER=app_user
MYSQL_PASSWORD=App@123456

# ============================================
# Redis配置
# ============================================
REDIS_PASSWORD=Redis@123456

# ============================================
# RabbitMQ配置
# ============================================
RABBITMQ_USER=admin
RABBITMQ_PASSWORD=Rabbit@123456

# ============================================
# Grafana配置
# ============================================
GRAFANA_USER=admin
GRAFANA_PASSWORD=Grafana@123456

# ============================================
# 应用安全配置
# ============================================
SECRET_KEY=your-super-secret-key-change-this-in-production
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# ============================================
# 应用配置
# ============================================
APP_ENV=production
DEBUG=false
LOG_LEVEL=INFO
```

### 4.2 开发环境覆盖

```yaml
# docker-compose.override.yml - 开发环境自动加载
version: '3.8'

services:
  app:
    environment:
      - APP_ENV=development
      - DEBUG=true
      - LOG_LEVEL=DEBUG
    volumes:
      - ./app:/app  # 代码热加载
    ports:
      - "5678:5678"  # debugpy端口

  mysql:
    ports:
      - "3306:3306"  # 暴露MySQL端口

  redis:
    ports:
      - "6379:6379"  # 暴露Redis端口

  elasticsearch:
    ports:
      - "9200:9200"  # 暴露ES端口
```

---

## 5. 配置文件

### 5.1 MySQL配置

```ini
# mysql/my.cnf
[mysqld]
# 基础配置
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
default-storage-engine = InnoDB

# 内存配置 (1GB总内存)
innodb_buffer_pool_size = 512M
innodb_log_buffer_size = 16M
innodb_log_file_size = 128M
key_buffer_size = 32M
query_cache_size = 32M
query_cache_limit = 4M
tmp_table_size = 32M
max_heap_table_size = 32M

# 连接配置
max_connections = 50
max_connect_errors = 1000
wait_timeout = 600
interactive_timeout = 600

# InnoDB优化
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1
innodb_io_capacity = 200

# 慢查询日志
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = 1

# 错误日志
log-error = /var/lib/mysql/error.log

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
```

### 5.2 Elasticsearch配置

```yaml
# es/elasticsearch.yml
cluster:
  name: data-platform

node:
  name: es-single
  roles: [master, data, ingest]

path:
  data: /usr/share/elasticsearch/data
  logs: /usr/share/elasticsearch/logs

network:
  host: 0.0.0.0
  bind_host: 0.0.0.0
  publish_host: elasticsearch

discovery:
  type: single-node

http:
  port: 9200
  host: 0.0.0.0
  cors:
    enabled: true
    allow-origin: "*"

transport:
  port: 9300

# 分片和副本(单节点)
index:
  number_of_shards: 1
  number_of_replicas: 0

# 内存优化
indices:
  memory:
    index_buffer_size: 30%
  fielddata:
    cache:
      size: 30%

# 磁盘水位线(160GB磁盘)
cluster:
  routing:
    allocation:
      disk:
        watermark:
          low: 85%
          high: 90%
          flood_stage: 95%

# 搜索优化
search:
  max_open_scroll_context: 100

# 禁用X-Pack安全(内网环境)
xpack:
  security:
    enabled: false
  monitoring:
    enabled: false
  watcher:
    enabled: false
```

### 5.3 Nginx配置

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    # 性能优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 100M;

    # Gzip压缩
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript 
               text/xml application/xml application/xml+rss text/javascript;

    # 限流配置
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=5r/m;
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    # Upstream配置
    upstream app {
        least_conn;
        server app:8000 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    # HTTP服务器
    server {
        listen 80;
        server_name localhost;
        
        # 安全响应头
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        # 前端静态文件
        location / {
            root /usr/share/nginx/html;
            try_files $uri $uri/ /index.html;
            expires 1d;
            add_header Cache-Control "public, immutable";
        }

        # API代理
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://app/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Connection "";
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }

        # WebSocket支持
        location /ws/ {
            proxy_pass http://app/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_read_timeout 86400s;
        }

        # 健康检查
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # Grafana代理(可选)
        location /grafana/ {
            proxy_pass http://grafana:3000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

### 5.4 Prometheus配置

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: data-platform
    replica: '{{.ExternalURL}}'

alerting:
  alertmanagers: []

rule_files:
  - 'alert_rules.yml'

scrape_configs:
  # Prometheus自身监控
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: /metrics

  # FastAPI应用监控
  - job_name: 'fastapi'
    static_configs:
      - targets: ['app:8000']
    metrics_path: /metrics
    scrape_interval: 10s

  # MySQL监控(需安装mysqld_exporter)
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql:9104']
    scrape_interval: 30s

  # Redis监控(需安装redis_exporter)
  - job_name: 'redis'
    static_configs:
      - targets: ['redis:9121']
    scrape_interval: 30s

  # Elasticsearch监控
  - job_name: 'elasticsearch'
    static_configs:
      - targets: ['elasticsearch:9200']
    metrics_path: /_prometheus/metrics
    scrape_interval: 30s

  # RabbitMQ监控
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15692']
    scrape_interval: 30s

  # cAdvisor容器监控(可选)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    scrape_interval: 30s

  # Node Exporter主机监控(可选)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    scrape_interval: 30s
```

```yaml
# prometheus/alert_rules.yml
groups:
  - name: system_alerts
    interval: 15s
    rules:
      # 内存告警
      - alert: HighMemoryUsage
        expr: |
          (
            1 - 
            (
              node_memory_MemAvailable_bytes 
              / 
              node_memory_MemTotal_bytes
            )
          ) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "内存使用率过高"
          description: "内存使用率超过85%，当前值: {{ $value }}%"

      # 磁盘告警
      - alert: HighDiskUsage
        expr: |
          (
            1 - 
            (
              node_filesystem_avail_bytes{mountpoint="/data"} 
              / 
              node_filesystem_size_bytes{mountpoint="/data"}
            )
          ) * 100 > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "磁盘使用率过高"
          description: "磁盘使用率超过85%，当前值: {{ $value }}%"

      # 应用响应时间
      - alert: AppHighLatency
        expr: |
          histogram_quantile(
            0.95, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "应用响应延迟过高"
          description: "P95响应时间超过1秒"

      # 错误率
      - alert: AppHighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) 
          / 
          rate(http_requests_total[5m]) * 100 > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "应用错误率过高"
          description: "5xx错误率超过5%"

      # MySQL连接数
      - alert: MySQLHighConnections
        expr: mysql_global_status_threads_connected > 40
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL连接数过高"
          description: "当前连接数: {{ $value }}"

      # Elasticsearch堆内存
      - alert: ESHighHeapUsage
        expr: |
          elasticsearch_jvm_memory_used_bytes{area="heap"} 
          / 
          elasticsearch_jvm_memory_max_bytes{area="heap"} * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "ES堆内存使用率过高"
          description: "当前使用率: {{ $value }}%"
```

---

## 6. 网络与存储

### 6.1 网络架构

```
┌─────────────────────────────────────────────────────────┐
│                      外部网络                            │
│                    (Internet/VPN)                       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   Nginx (80/443)                        │
│              反向代理 + 负载均衡 + SSL                    │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌──────────┼──────────┐
         │          │          │
         ▼          ▼          ▼
┌─────────────┐ ┌──────────┐ ┌─────────────┐
│ FastAPI     │ │ Grafana  │ │ Prometheus  │
│ :8000       │ │ :3000    │ │ :9090       │
└─────────────┘ └──────────┘ └─────────────┘
         │
         ├─────────────────────────────────┐
         │           内部网络(backend)       │
         ├─────────────────────────────────┤
         │                                 │
         ▼                                 ▼
┌─────────────┐  ┌──────────┐  ┌─────────────────────┐
│   MySQL     │  │  Redis   │  │   Elasticsearch     │
│   :3306     │  │  :6379   │  │      :9200          │
└─────────────┘  └──────────┘  └─────────────────────┘
         │                                 │
         ▼                                 ▼
┌─────────────────────────────────────────────────────────┐
│                  RabbitMQ (:5672)                       │
│              消息队列 (Celery Backend)                   │
└─────────────────────────────────────────────────────────┘
```

### 6.2 存储映射

| 服务 | 容器路径 | 宿主机路径 | 说明 |
|------|----------|------------|------|
| MySQL | /var/lib/mysql | ./mysql/data | 数据库文件 |
| Redis | /data | ./redis/data | RDB/AOF文件 |
| Elasticsearch | /usr/share/elasticsearch/data | ./es/data | 索引数据 |
| RabbitMQ | /var/lib/rabbitmq | ./rabbitmq/data | 消息队列数据 |
| Prometheus | /prometheus | ./prometheus/data | 监控数据 |
| Grafana | /var/lib/grafana | ./grafana/data | 配置和仪表板 |
| Nginx | /var/log/nginx | ./logs/nginx | 访问/错误日志 |
| App | /app/logs | ./logs | 应用日志 |

---

## 7. 启动与运维

### 7.1 启动步骤

```bash
#!/bin/bash
# start.sh - 启动脚本

set -e

echo "=== 数据平台启动脚本 ==="

# 1. 创建目录
echo "[1/6] 创建数据目录..."
mkdir -p /data/{mysql,redis,es,rabbitmq,prometheus,grafana,nginx,logs,backups,scripts}
chmod 777 /data/es/data

# 2. 检查环境变量
echo "[2/6] 检查环境变量..."
if [ ! -f /data/.env ]; then
    echo "错误: .env 文件不存在"
    exit 1
fi

# 3. 拉取镜像
echo "[3/6] 拉取镜像..."
cd /data
docker-compose pull

# 4. 启动基础服务
echo "[4/6] 启动基础服务 (MySQL, Redis, ES, RabbitMQ)..."
docker-compose up -d mysql redis elasticsearch rabbitmq

# 5. 等待服务就绪
echo "[5/6] 等待服务就绪..."
sleep 10

# 检查MySQL
until docker exec mysql mysqladmin ping -h localhost -u root -p"${MYSQL_ROOT_PASSWORD}" --silent; do
    echo "等待MySQL..."
    sleep 2
done

# 检查ES
until curl -s http://localhost:9200/_cluster/health | grep -q '"status":"green\|yellow"'; do
    echo "等待Elasticsearch..."
    sleep 2
done

# 6. 启动应用服务
echo "[6/6] 启动应用服务..."
docker-compose up -d app celery-worker celery-beat prometheus grafana nginx

echo "=== 启动完成 ==="
echo "访问地址:"
echo "  - Web界面: http://localhost"
echo "  - API文档: http://localhost/api/docs"
echo "  - Grafana: http://localhost:3000"
echo "  - RabbitMQ管理: http://localhost:15672"
```

### 7.2 常用运维命令

```bash
# 查看所有服务状态
docker-compose ps

# 查看资源使用
docker stats --no-stream

# 查看日志
docker-compose logs -f app
docker-compose logs -f celery-worker
docker-compose logs -f mysql

# 重启服务
docker-compose restart app
docker-compose restart celery-worker

# 进入容器
docker exec -it data-platform-app /bin/sh
docker exec -it mysql mysql -u root -p

# 扩展worker
docker-compose up -d --scale celery-worker=2

# 备份
docker exec mysql mysqldump -u root -p data_platform | gzip > backup.sql.gz

# 清理
docker system prune -f
docker volume prune -f
```

### 7.3 故障排查

```bash
# 服务无法启动 - 查看日志
docker-compose logs SERVICE_NAME

# 内存不足
dmesg | grep -i "out of memory"
docker system df

# MySQL连接问题
docker exec mysql mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
docker exec mysql mysql -u root -p -e "SHOW PROCESSLIST;"

# ES问题
curl http://localhost:9200/_cluster/health
curl http://localhost:9200/_cat/indices?v

# 磁盘空间
df -h
du -sh /data/* | sort -rh
```

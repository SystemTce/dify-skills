# 监控和告警

## 概述

完善的监控和告警系统是生产环境的关键组成部分,帮助及时发现和解决问题。

## 监控架构

```
┌─────────────────────────────────────────┐
│          Dify Services                  │
│  (API, Worker, Web, Sandbox)            │
└────────┬────────────────────────────────┘
         │ Metrics & Traces
         ↓
┌─────────────────────────────────────────┐
│     OpenTelemetry Collector             │
└────────┬────────────────────────────────┘
         │
         ├──→ Prometheus (Metrics)
         ├──→ Jaeger (Traces)
         └──→ Loki (Logs)
                  ↓
         ┌────────────────┐
         │    Grafana     │ ← 可视化
         └────────────────┘
```

## OpenTelemetry 配置

### 环境变量

```bash
# .env
ENABLE_OTEL=true
OTLP_BASE_ENDPOINT=http://otel-collector:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_TYPE=otlp
OTEL_SAMPLING_RATE=0.1

# 批量导出配置
OTEL_BATCH_EXPORT_SCHEDULE_DELAY=5000
OTEL_MAX_QUEUE_SIZE=2048
OTEL_MAX_EXPORT_BATCH_SIZE=512
OTEL_BATCH_EXPORT_TIMEOUT=10000

# 指标导出配置
OTEL_METRIC_EXPORT_INTERVAL=60000
OTEL_METRIC_EXPORT_TIMEOUT=30000
```

### OpenTelemetry Collector 配置

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"

  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger]

    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]

    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

### Docker Compose 集成

```yaml
# docker-compose.monitoring.yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"  # OTLP gRPC
      - "4318:4318"  # OTLP HTTP
      - "8889:8889"  # Prometheus metrics

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "14250:14250"  # gRPC

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

volumes:
  prometheus_data:
  grafana_data:
```

## Sentry 错误追踪

### 配置

```bash
# .env
# API Service
API_SENTRY_DSN=https://your-key@sentry.io/project-id
API_SENTRY_TRACES_SAMPLE_RATE=1.0
API_SENTRY_PROFILES_SAMPLE_RATE=1.0

# Web Service
WEB_SENTRY_DSN=https://your-key@sentry.io/project-id

# Plugin Service
PLUGIN_SENTRY_ENABLED=true
PLUGIN_SENTRY_DSN=https://your-key@sentry.io/project-id
```

### 采样率配置

```bash
# 生产环境 - 降低采样率
API_SENTRY_TRACES_SAMPLE_RATE=0.1  # 10% 采样
API_SENTRY_PROFILES_SAMPLE_RATE=0.1

# 开发环境 - 全量采样
API_SENTRY_TRACES_SAMPLE_RATE=1.0  # 100% 采样
```

## 关键监控指标

### 应用指标

```yaml
# API 服务指标
- http_request_duration_seconds  # 请求延迟
- http_requests_total            # 请求总数
- http_request_errors_total      # 错误总数
- active_requests                # 活跃请求数

# Worker 指标
- celery_task_duration_seconds   # 任务执行时间
- celery_tasks_total             # 任务总数
- celery_task_failures_total     # 任务失败数
- celery_queue_length            # 队列长度

# 数据库指标
- db_connections_active          # 活跃连接数
- db_connections_idle            # 空闲连接数
- db_query_duration_seconds      # 查询延迟
```

### 系统指标

```yaml
# 资源使用
- container_cpu_usage_seconds_total
- container_memory_usage_bytes
- container_network_receive_bytes_total
- container_network_transmit_bytes_total

# 磁盘使用
- node_filesystem_avail_bytes
- node_filesystem_size_bytes
```

### Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8889']

  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

## 队列监控

### 配置

```bash
# .env
QUEUE_MONITOR_THRESHOLD=200
QUEUE_MONITOR_ALERT_EMAILS=admin@example.com,ops@example.com
QUEUE_MONITOR_INTERVAL=30
ENABLE_DATASETS_QUEUE_MONITOR=true
```

### 监控脚本

```python
# queue_monitor.py
import redis
import smtplib
from email.mime.text import MIMEText

def check_queue_depth():
    r = redis.Redis(host='redis', port=6379, password='your-password')

    # 检查 Celery 队列
    queue_length = r.llen('celery')

    if queue_length > 200:
        send_alert(f"Queue depth high: {queue_length}")

    return queue_length

def send_alert(message):
    msg = MIMEText(message)
    msg['Subject'] = 'Dify Queue Alert'
    msg['From'] = 'monitor@example.com'
    msg['To'] = 'admin@example.com'

    s = smtplib.SMTP('localhost')
    s.send_message(msg)
    s.quit()

if __name__ == '__main__':
    check_queue_depth()
```

## 日志管理

### 日志配置

```bash
# .env
LOG_LEVEL=INFO
LOG_OUTPUT_FORMAT=json
LOG_FILE=/app/logs/server.log
LOG_FILE_MAX_SIZE=20
LOG_FILE_BACKUP_COUNT=5
LOG_DATEFORMAT=%Y-%m-%d %H:%M:%S
LOG_TZ=UTC
```

### Loki 配置

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h

storage_config:
  boltdb:
    directory: /loki/index
  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
```

### Promtail 配置

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
```

## Grafana 仪表板

### 数据源配置

```yaml
# grafana/datasources/datasources.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100

  - name: Jaeger
    type: jaeger
    access: proxy
    url: http://jaeger:16686
```

### 仪表板示例

```json
{
  "dashboard": {
    "title": "Dify Overview",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_request_errors_total[5m])"
          }
        ]
      },
      {
        "title": "Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Queue Depth",
        "targets": [
          {
            "expr": "celery_queue_length"
          }
        ]
      }
    ]
  }
}
```

## 告警规则

### Prometheus 告警

```yaml
# prometheus-alerts.yaml
groups:
  - name: dify_alerts
    interval: 30s
    rules:
      # 高错误率
      - alert: HighErrorRate
        expr: rate(http_request_errors_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors/sec"

      # 高响应时间
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s"

      # 队列积压
      - alert: QueueBacklog
        expr: celery_queue_length > 200
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Queue backlog detected"
          description: "Queue length is {{ $value }}"

      # 数据库连接池耗尽
      - alert: DatabaseConnectionPoolExhausted
        expr: db_connections_active / db_connections_max > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool nearly exhausted"
          description: "{{ $value }}% of connections in use"

      # 磁盘空间不足
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low"
          description: "Only {{ $value }}% disk space remaining"
```

### Alertmanager 配置

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'email'

receivers:
  - name: 'email'
    email_configs:
      - to: 'admin@example.com'
        headers:
          Subject: 'Dify Alert: {{ .GroupLabels.alertname }}'

  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'
```

## 健康检查

### 端点配置

```python
# health_check.py
from flask import Flask, jsonify
import redis
import psycopg2

app = Flask(__name__)

@app.route('/health')
def health():
    checks = {
        'database': check_database(),
        'redis': check_redis(),
        'disk': check_disk_space()
    }

    status = 'healthy' if all(checks.values()) else 'unhealthy'

    return jsonify({
        'status': status,
        'checks': checks
    }), 200 if status == 'healthy' else 503

def check_database():
    try:
        conn = psycopg2.connect(
            host='db_postgres',
            database='dify',
            user='postgres',
            password='password'
        )
        conn.close()
        return True
    except:
        return False

def check_redis():
    try:
        r = redis.Redis(host='redis', port=6379)
        r.ping()
        return True
    except:
        return False

def check_disk_space():
    import shutil
    stat = shutil.disk_usage('/')
    return stat.free / stat.total > 0.1
```

### 监控脚本

```bash
#!/bin/bash
# health_monitor.sh

ENDPOINT="http://localhost/health"
ALERT_EMAIL="admin@example.com"

while true; do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" $ENDPOINT)

    if [ $STATUS -ne 200 ]; then
        echo "Health check failed with status $STATUS"
        echo "Health check failed" | mail -s "Dify Health Alert" $ALERT_EMAIL
    fi

    sleep 60
done
```

## 性能监控

### 应用性能监控 (APM)

```bash
# 使用 Sentry Performance Monitoring
API_SENTRY_TRACES_SAMPLE_RATE=0.1
API_SENTRY_PROFILES_SAMPLE_RATE=0.1
```

### 数据库性能监控

```sql
-- 慢查询监控
SELECT
    query,
    calls,
    total_time,
    mean_time,
    max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- 连接数监控
SELECT count(*) FROM pg_stat_activity;

-- 锁监控
SELECT * FROM pg_locks WHERE NOT granted;
```

### Redis 性能监控

```bash
# Redis 信息
docker compose exec redis redis-cli INFO

# 内存使用
docker compose exec redis redis-cli INFO memory

# 慢查询
docker compose exec redis redis-cli SLOWLOG GET 10
```

## 参考资料

- [OpenTelemetry 文档](https://opentelemetry.io/docs/)
- [Prometheus 文档](https://prometheus.io/docs/)
- [Grafana 文档](https://grafana.com/docs/)
- [Sentry 文档](https://docs.sentry.io/)

---

**最后更新**: 2026-03-04

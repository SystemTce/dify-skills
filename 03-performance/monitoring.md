# 性能监控和分析完整指南

## 概述

性能监控是持续优化的基础。本文档提供完整的监控策略、工具和最佳实践。

## 监控架构

```
应用层 → 指标收集 → 时序数据库 → 可视化 → 告警
  ↓         ↓           ↓          ↓       ↓
Dify    Prometheus   Prometheus  Grafana  Alert
        Exporter                          Manager
```

## 核心指标体系

### 1. 工作流执行指标

**关键指标**:

| 指标名称 | 类型 | 目标值 | 说明 |
|---------|------|-------|------|
| workflow_execution_total | Counter | - | 工作流执行总数 |
| workflow_execution_duration_seconds | Histogram | P95 < 5s | 执行时间分布 |
| workflow_execution_success_rate | Gauge | > 99% | 成功率 |
| workflow_node_execution_total | Counter | - | 节点执行总数 |
| workflow_node_failure_total | Counter | < 1% | 节点失败数 |

**Prometheus 查询**:

```promql
# 工作流执行速率 (QPS)
rate(workflow_execution_total[5m])

# P95 执行时间
histogram_quantile(0.95, rate(workflow_execution_duration_seconds_bucket[5m]))

# 成功率
sum(rate(workflow_execution_total{status="success"}[5m])) /
sum(rate(workflow_execution_total[5m])) * 100

# 节点失败率
sum(rate(workflow_node_failure_total[5m])) /
sum(rate(workflow_node_execution_total[5m])) * 100
```

### 2. LLM 使用指标

**关键指标**:

| 指标名称 | 类型 | 目标值 | 说明 |
|---------|------|-------|------|
| llm_api_calls_total | Counter | - | LLM API 调用总数 |
| llm_api_duration_seconds | Histogram | P95 < 3s | API 响应时间 |
| llm_token_usage_total | Counter | < 预算 | Token 使用总数 |
| llm_api_cost_total | Counter | < 预算 | API 成本总计 |
| llm_cache_hit_rate | Gauge | > 30% | 缓存命中率 |

**Prometheus 查询**:

```promql
# LLM API 调用速率
rate(llm_api_calls_total[5m])

# Token 使用速率
rate(llm_token_usage_total[5m])

# 每日成本
increase(llm_api_cost_total[24h])

# 缓存命中率
sum(rate(llm_cache_hits_total[5m])) /
sum(rate(llm_cache_requests_total[5m])) * 100
```

### 3. 系统资源指标

**关键指标**:

| 指标名称 | 类型 | 目标值 | 说明 |
|---------|------|-------|------|
| system_cpu_usage_percent | Gauge | < 70% | CPU 使用率 |
| system_memory_usage_percent | Gauge | < 80% | 内存使用率 |
| system_disk_usage_percent | Gauge | < 80% | 磁盘使用率 |
| database_connections_active | Gauge | < 80% | 数据库连接数 |
| redis_memory_usage_bytes | Gauge | < 70% | Redis 内存使用 |

**Prometheus 查询**:

```promql
# CPU 使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 数据库连接使用率
pg_stat_database_numbackends / pg_settings_max_connections * 100
```

### 4. Worker 池指标

**关键指标**:

| 指标名称 | 类型 | 目标值 | 说明 |
|---------|------|-------|------|
| worker_pool_size | Gauge | 2-8 | Worker 数量 |
| worker_pool_queue_depth | Gauge | < 10 | 队列深度 |
| worker_pool_utilization | Gauge | 60-80% | Worker 利用率 |
| worker_pool_idle_time_seconds | Histogram | - | Worker 空闲时间 |

**Prometheus 查询**:

```promql
# Worker 利用率
worker_pool_active_workers / worker_pool_size * 100

# 平均队列深度
avg_over_time(worker_pool_queue_depth[5m])

# Worker 扩缩容事件
rate(worker_pool_scale_events_total[5m])
```

## 监控实现

### 1. Prometheus Exporter

**实现示例**:

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# 定义指标
workflow_execution_total = Counter(
    'workflow_execution_total',
    'Total workflow executions',
    ['workflow_id', 'status']
)

workflow_execution_duration = Histogram(
    'workflow_execution_duration_seconds',
    'Workflow execution duration',
    ['workflow_id'],
    buckets=[0.1, 0.5, 1, 2, 5, 10, 30, 60]
)

llm_token_usage = Counter(
    'llm_token_usage_total',
    'Total LLM token usage',
    ['model', 'type']  # type: input/output
)

worker_pool_size = Gauge(
    'worker_pool_size',
    'Current worker pool size'
)

# 记录指标
def record_workflow_execution(workflow_id, duration, status):
    workflow_execution_total.labels(
        workflow_id=workflow_id,
        status=status
    ).inc()

    workflow_execution_duration.labels(
        workflow_id=workflow_id
    ).observe(duration)

def record_llm_usage(model, input_tokens, output_tokens):
    llm_token_usage.labels(model=model, type='input').inc(input_tokens)
    llm_token_usage.labels(model=model, type='output').inc(output_tokens)

# 启动 HTTP 服务器
if __name__ == '__main__':
    start_http_server(8000)  # 在 8000 端口暴露指标
    while True:
        time.sleep(1)
```

### 2. Grafana 仪表板

**仪表板配置**:

```json
{
  "dashboard": {
    "title": "Dify Performance Dashboard",
    "panels": [
      {
        "title": "Workflow Execution Rate",
        "targets": [
          {
            "expr": "rate(workflow_execution_total[5m])"
          }
        ],
        "type": "graph"
      },
      {
        "title": "P95 Execution Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(workflow_execution_duration_seconds_bucket[5m]))"
          }
        ],
        "type": "graph"
      },
      {
        "title": "LLM Token Usage",
        "targets": [
          {
            "expr": "rate(llm_token_usage_total[5m])"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Worker Pool Size",
        "targets": [
          {
            "expr": "worker_pool_size"
          }
        ],
        "type": "gauge"
      }
    ]
  }
}
```

### 3. 告警规则

**Prometheus 告警配置**:

```yaml
groups:
  - name: dify_alerts
    rules:
      # 工作流执行失败率过高
      - alert: HighWorkflowFailureRate
        expr: |
          sum(rate(workflow_execution_total{status="failure"}[5m])) /
          sum(rate(workflow_execution_total[5m])) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "工作流失败率过高"
          description: "失败率: {{ $value | humanizePercentage }}"

      # LLM API 响应时间过长
      - alert: SlowLLMResponse
        expr: |
          histogram_quantile(0.95, rate(llm_api_duration_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "LLM API 响应时间过长"
          description: "P95 响应时间: {{ $value }}s"

      # Worker 队列深度过大
      - alert: HighWorkerQueueDepth
        expr: worker_pool_queue_depth > 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Worker 队列深度过大"
          description: "队列深度: {{ $value }}"

      # 内存使用率过高
      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "内存使用率过高"
          description: "内存使用率: {{ $value | humanizePercentage }}"
```

## 日志分析

### 1. 结构化日志

**日志格式**:

```python
import logging
import json

class StructuredLogger:
    def __init__(self, name):
        self.logger = logging.getLogger(name)

    def log_workflow_execution(self, workflow_id, duration, status, **kwargs):
        log_data = {
            "event": "workflow_execution",
            "workflow_id": workflow_id,
            "duration": duration,
            "status": status,
            "timestamp": time.time(),
            **kwargs
        }
        self.logger.info(json.dumps(log_data))

    def log_llm_call(self, model, prompt_tokens, completion_tokens, cost):
        log_data = {
            "event": "llm_call",
            "model": model,
            "prompt_tokens": prompt_tokens,
            "completion_tokens": completion_tokens,
            "cost": cost,
            "timestamp": time.time()
        }
        self.logger.info(json.dumps(log_data))
```

### 2. 日志聚合

**ELK Stack 配置**:

```yaml
# Logstash 配置
input {
  file {
    path => "/var/log/dify/*.log"
    codec => json
  }
}

filter {
  if [event] == "workflow_execution" {
    mutate {
      add_field => { "[@metadata][index]" => "workflow-executions" }
    }
  }

  if [event] == "llm_call" {
    mutate {
      add_field => { "[@metadata][index]" => "llm-calls" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "%{[@metadata][index]}-%{+YYYY.MM.dd}"
  }
}
```

### 3. 日志查询

**Elasticsearch 查询**:

```json
{
  "query": {
    "bool": {
      "must": [
        { "term": { "event": "workflow_execution" } },
        { "term": { "status": "failure" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "failure_reasons": {
      "terms": { "field": "error_type" }
    }
  }
}
```

## 性能分析工具

### 1. Python Profiler

**使用示例**:

```python
import cProfile
import pstats
from pstats import SortKey

def profile_workflow(workflow_id):
    profiler = cProfile.Profile()
    profiler.enable()

    # 执行工作流
    run_workflow(workflow_id)

    profiler.disable()

    # 生成报告
    stats = pstats.Stats(profiler)
    stats.sort_stats(SortKey.CUMULATIVE)
    stats.print_stats(20)  # 显示前 20 个最耗时的函数

    # 保存报告
    stats.dump_stats(f"profile_{workflow_id}.prof")
```

### 2. 分布式追踪

**OpenTelemetry 集成**:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

# 配置追踪
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)

trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# 使用追踪
def run_workflow_with_tracing(workflow_id):
    with tracer.start_as_current_span("workflow_execution") as span:
        span.set_attribute("workflow_id", workflow_id)

        with tracer.start_as_current_span("node_execution"):
            execute_node()

        with tracer.start_as_current_span("llm_call"):
            call_llm()
```

## 性能报告

### 1. 日报生成

**自动化脚本**:

```python
def generate_daily_report(date):
    """生成每日性能报告"""
    report = {
        "date": date,
        "workflow_executions": query_workflow_executions(date),
        "llm_usage": query_llm_usage(date),
        "cost": query_daily_cost(date),
        "errors": query_errors(date),
        "performance": {
            "avg_execution_time": query_avg_execution_time(date),
            "p95_execution_time": query_p95_execution_time(date),
            "success_rate": query_success_rate(date)
        }
    }

    # 生成 HTML 报告
    html = render_report_template(report)

    # 发送邮件
    send_email(
        to="team@example.com",
        subject=f"Dify Performance Report - {date}",
        html=html
    )

    return report
```

### 2. 趋势分析

**周报分析**:

```python
def analyze_weekly_trends():
    """分析每周性能趋势"""
    last_7_days = get_last_7_days()

    trends = {
        "execution_time": [],
        "success_rate": [],
        "cost": []
    }

    for date in last_7_days:
        trends["execution_time"].append(
            query_avg_execution_time(date)
        )
        trends["success_rate"].append(
            query_success_rate(date)
        )
        trends["cost"].append(
            query_daily_cost(date)
        )

    # 计算趋势
    execution_time_trend = calculate_trend(trends["execution_time"])
    success_rate_trend = calculate_trend(trends["success_rate"])
    cost_trend = calculate_trend(trends["cost"])

    return {
        "execution_time_trend": execution_time_trend,
        "success_rate_trend": success_rate_trend,
        "cost_trend": cost_trend
    }
```

## 最佳实践

1. **建立基线**: 记录优化前的性能基线
2. **持续监控**: 24/7 监控关键指标
3. **及时告警**: 设置合理的告警阈值
4. **定期分析**: 每周分析性能趋势
5. **优化迭代**: 根据监控数据持续优化

## 监控清单

- [ ] 部署 Prometheus 和 Grafana
- [ ] 配置 Prometheus Exporter
- [ ] 创建 Grafana 仪表板
- [ ] 设置告警规则
- [ ] 配置日志聚合 (ELK/Loki)
- [ ] 集成分布式追踪 (Jaeger/Zipkin)
- [ ] 建立性能基线
- [ ] 设置自动化报告

## 参考资料

- [Prometheus 官方文档](https://prometheus.io/docs/)
- [Grafana 官方文档](https://grafana.com/docs/)
- [OpenTelemetry 文档](https://opentelemetry.io/docs/)
- [ELK Stack 文档](https://www.elastic.co/guide/)

---

**最后更新**: 2026-03-04

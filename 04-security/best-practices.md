# 安全和部署最佳实践

## 概述

本文档总结 Dify 安全和部署的最佳实践,帮助构建安全、稳定、高性能的生产环境。

---

## 安全最佳实践

### 1. 密钥管理

#### 强密钥生成

```bash
# 生成强随机密钥
SECRET_KEY=$(openssl rand -base64 42)
SANDBOX_API_KEY=$(openssl rand -base64 32)
PLUGIN_DAEMON_KEY=$(openssl rand -base64 42)

# 数据库密码
DB_PASSWORD=$(openssl rand -base64 24)
REDIS_PASSWORD=$(openssl rand -base64 24)
```

#### 密钥轮换

```bash
# 定期轮换密钥 (建议每 90 天)
# 1. 生成新密钥
NEW_SECRET_KEY=$(openssl rand -base64 42)

# 2. 更新 .env
sed -i "s/SECRET_KEY=.*/SECRET_KEY=$NEW_SECRET_KEY/" .env

# 3. 重启服务
docker compose restart api worker

# 4. 验证
curl http://localhost/health
```

#### 密钥存储

```bash
# ❌ 不要将密钥提交到 Git
echo ".env" >> .gitignore
echo "*.key" >> .gitignore

# ✅ 使用密钥管理服务
# AWS Secrets Manager
aws secretsmanager get-secret-value --secret-id dify/secret-key

# HashiCorp Vault
vault kv get secret/dify/secret-key

# Kubernetes Secrets
kubectl create secret generic dify-secrets \
  --from-literal=secret-key=$SECRET_KEY \
  --from-literal=db-password=$DB_PASSWORD
```

### 2. 网络安全

#### SSRF 防护

```conf
# squid.conf - 严格的 ACL 规则
# 1. 定义内网地址
acl localnet src 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 127.0.0.0/8

# 2. 定义允许的域名 (白名单模式)
acl allowed_domains dstdomain .openai.com .anthropic.com

# 3. 拒绝内网访问
http_access deny localnet

# 4. 仅允许白名单
http_access allow allowed_domains
http_access deny all
```

#### 网络隔离

```yaml
# docker-compose.yaml
networks:
  # 内部网络 - 无法访问外部
  ssrf_proxy_network:
    driver: bridge
    internal: true

  # 默认网络 - 可访问外部
  default:
    driver: bridge

services:
  # 敏感服务使用内部网络
  sandbox:
    networks:
      - ssrf_proxy_network

  # 代理服务连接两个网络
  ssrf_proxy:
    networks:
      - ssrf_proxy_network
      - default
```

#### TLS/HTTPS 配置

```bash
# 使用 Certbot 自动管理证书
docker compose --profile certbot up -d
docker compose exec certbot /bin/sh /update-cert.sh

# 或使用自签名证书 (仅开发环境)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/dify.key \
  -out nginx/ssl/dify.crt
```

### 3. 访问控制

#### 最小权限原则

```yaml
# docker-compose.yaml - 以非 root 用户运行
services:
  api:
    user: "1001:1001"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

#### API 认证

```bash
# 启用 API Key 认证
SECRET_KEY=<strong-random-key>

# 配置 OAuth 2.0
OAUTH_ENABLED=true
OAUTH_CLIENT_ID=<your-client-id>
OAUTH_CLIENT_SECRET=<your-client-secret>
```

#### 角色权限

```python
# 实施基于角色的访问控制 (RBAC)
roles = {
    'admin': ['read', 'write', 'delete', 'manage'],
    'developer': ['read', 'write'],
    'viewer': ['read']
}
```

### 4. 数据保护

#### 数据加密

```bash
# 传输加密
NGINX_HTTPS_ENABLED=true
NGINX_SSL_PROTOCOLS=TLSv1.2 TLSv1.3

# 数据库连接加密
DB_SSL_MODE=require
REDIS_USE_SSL=true

# 存储加密
# 使用云厂商的加密存储服务
S3_SERVER_SIDE_ENCRYPTION=AES256
```

#### 敏感数据处理

```python
# 不要记录敏感信息
import logging

# ❌ 错误
logging.info(f"User password: {password}")

# ✅ 正确
logging.info(f"User authenticated: {user_id}")

# 使用环境变量存储敏感配置
import os
api_key = os.getenv('OPENAI_API_KEY')
```

### 5. 审计和监控

#### 访问日志

```bash
# 启用详细日志
LOG_LEVEL=INFO
ENABLE_REQUEST_LOGGING=True

# 日志格式
LOG_OUTPUT_FORMAT=json
LOG_DATEFORMAT=%Y-%m-%d %H:%M:%S
```

#### 安全审计

```bash
# 定期审计脚本
#!/bin/bash

# 1. 检查弱密码
if grep -q "difyai123456" .env; then
    echo "Warning: Default password detected"
fi

# 2. 检查 HTTPS
if ! grep -q "NGINX_HTTPS_ENABLED=true" .env; then
    echo "Warning: HTTPS not enabled"
fi

# 3. 检查密钥强度
SECRET_KEY=$(grep SECRET_KEY .env | cut -d= -f2)
if [ ${#SECRET_KEY} -lt 32 ]; then
    echo "Warning: Weak SECRET_KEY"
fi
```

---

## 部署最佳实践

### 1. 环境分离

#### 开发环境

```bash
# .env.development
DEBUG=true
LOG_LEVEL=DEBUG
DEPLOY_ENV=DEVELOPMENT

# 使用默认密码
DB_PASSWORD=difyai123456
REDIS_PASSWORD=difyai123456

# 禁用 HTTPS
NGINX_HTTPS_ENABLED=false
```

#### 生产环境

```bash
# .env.production
DEBUG=false
LOG_LEVEL=INFO
DEPLOY_ENV=PRODUCTION

# 强密钥
SECRET_KEY=$(openssl rand -base64 42)
DB_PASSWORD=<strong-password>
REDIS_PASSWORD=<strong-password>

# 启用 HTTPS
NGINX_HTTPS_ENABLED=true

# 启用监控
ENABLE_OTEL=true
API_SENTRY_DSN=<your-sentry-dsn>
```

### 2. 资源配置

#### 合理的资源限制

```yaml
# docker-compose.yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

  worker:
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
```

#### 根据负载调整

```bash
# 低负载 (< 100 用户)
API_REPLICAS=1
WORKER_REPLICAS=1
CELERY_WORKER_AMOUNT=2

# 中负载 (100-1000 用户)
API_REPLICAS=2
WORKER_REPLICAS=2
CELERY_WORKER_AMOUNT=4

# 高负载 (> 1000 用户)
API_REPLICAS=3
WORKER_REPLICAS=3
CELERY_WORKER_AMOUNT=8
```

### 3. 高可用配置

#### 多副本部署

```yaml
# docker-compose.ha.yaml
services:
  api:
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        max_attempts: 3

  worker:
    deploy:
      replicas: 2
```

#### 健康检查

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:5001/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

#### 负载均衡

```nginx
# nginx.conf
upstream api_backend {
    least_conn;
    server api1:5001 max_fails=3 fail_timeout=30s;
    server api2:5001 max_fails=3 fail_timeout=30s;
    server api3:5001 max_fails=3 fail_timeout=30s;
}

server {
    location /api {
        proxy_pass http://api_backend;
    }
}
```

### 4. 备份策略

#### 自动备份

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR=/backups
DATE=$(date +%Y%m%d_%H%M%S)

# 1. 备份数据库
docker compose exec -T db_postgres pg_dump -U postgres dify | \
    gzip > $BACKUP_DIR/db_$DATE.sql.gz

# 2. 备份文件存储
tar -czf $BACKUP_DIR/storage_$DATE.tar.gz ./volumes/app/storage

# 3. 备份配置
cp .env $BACKUP_DIR/env_$DATE

# 4. 上传到云存储
aws s3 cp $BACKUP_DIR/ s3://dify-backups/$DATE/ --recursive

# 5. 清理旧备份 (保留 30 天)
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete
```

#### 定时任务

```bash
# crontab -e
# 每天凌晨 2 点备份
0 2 * * * /path/to/backup.sh

# 每周日凌晨 3 点完整备份
0 3 * * 0 /path/to/full_backup.sh
```

#### 恢复测试

```bash
# 定期测试恢复流程
#!/bin/bash

# 1. 恢复数据库
gunzip < backup.sql.gz | \
    docker compose exec -T db_postgres psql -U postgres dify

# 2. 恢复文件
tar -xzf storage_backup.tar.gz -C ./volumes/app/

# 3. 验证
docker compose restart api
curl http://localhost/health
```

### 5. 监控和告警

#### 关键指标

```yaml
# 应用指标
- http_request_duration_seconds  # 响应时间
- http_requests_total            # 请求总数
- http_request_errors_total      # 错误总数
- celery_queue_length            # 队列深度

# 系统指标
- container_cpu_usage_seconds_total
- container_memory_usage_bytes
- node_filesystem_avail_bytes

# 业务指标
- active_users                   # 活跃用户数
- workflow_executions_total      # 工作流执行次数
- llm_api_calls_total           # LLM API 调用次数
```

#### 告警规则

```yaml
# prometheus-alerts.yaml
groups:
  - name: critical_alerts
    rules:
      # 服务不可用
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical

      # 高错误率
      - alert: HighErrorRate
        expr: rate(http_request_errors_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning

      # 队列积压
      - alert: QueueBacklog
        expr: celery_queue_length > 200
        for: 10m
        labels:
          severity: critical
```

### 6. 更新和维护

#### 滚动更新

```bash
# 1. 拉取最新镜像
docker compose pull

# 2. 逐个更新服务
docker compose up -d --no-deps --build api
sleep 30
docker compose up -d --no-deps --build worker

# 3. 验证
docker compose ps
curl http://localhost/health
```

#### 版本管理

```bash
# 使用特定版本
# docker-compose.yaml
services:
  api:
    image: langgenius/dify-api:1.13.0  # 固定版本

# 或使用标签
  api:
    image: langgenius/dify-api:stable
```

#### 回滚策略

```bash
# 1. 保存当前版本
docker compose config > docker-compose.backup.yaml

# 2. 如果更新失败,回滚
docker compose -f docker-compose.backup.yaml up -d

# 3. 验证
docker compose ps
```

---

## 运维最佳实践

### 1. 文档维护

#### 运维手册

```markdown
# 运维手册

## 服务器信息
- 主机: production-server-01
- IP: 10.0.1.100
- 配置: 8 vCPU, 32GB RAM

## 部署信息
- 版本: 1.13.0
- 部署日期: 2026-03-04
- 负责人: ops@example.com

## 常用命令
- 启动: docker compose up -d
- 停止: docker compose down
- 查看日志: docker compose logs -f api

## 应急联系
- 技术负责人: tech-lead@example.com
- 运维团队: ops@example.com
- 紧急电话: +86-xxx-xxxx-xxxx
```

#### 变更记录

```markdown
# 变更记录

## 2026-03-04
- 升级到 1.13.0
- 增加 Worker 副本数到 3
- 启用 OpenTelemetry 监控

## 2026-03-01
- 修复数据库连接池配置
- 更新 SSL 证书
```

### 2. 容量规划

#### 资源监控

```bash
# 监控资源使用趋势
docker stats --no-stream > stats_$(date +%Y%m%d).log

# 分析趋势
awk '{print $3}' stats_*.log | sort -n | tail -10
```

#### 扩容决策

```bash
# CPU 使用率 > 80% 持续 5 分钟 → 扩容
# 内存使用率 > 85% → 扩容
# 队列深度 > 200 持续 10 分钟 → 增加 Worker

# 扩容命令
docker compose up -d --scale worker=5
```

### 3. 故障演练

#### 定期演练

```bash
# 1. 数据库故障演练
docker compose stop db_postgres
# 观察服务行为
docker compose start db_postgres

# 2. 网络故障演练
docker network disconnect dify_default api
# 观察服务行为
docker network connect dify_default api

# 3. 高负载演练
# 使用压测工具模拟高负载
ab -n 10000 -c 100 http://localhost/api/health
```

#### 应急响应

```markdown
# 应急响应流程

## 1. 发现问题
- 监控告警
- 用户反馈
- 日志异常

## 2. 初步诊断
- 查看服务状态
- 查看日志
- 查看资源使用

## 3. 快速恢复
- 重启服务
- 回滚版本
- 切换备用系统

## 4. 根因分析
- 详细日志分析
- 性能分析
- 代码审查

## 5. 预防措施
- 修复问题
- 更新文档
- 改进监控
```

---

## 检查清单

### 部署前检查

- [ ] 所有密钥已更改为强随机值
- [ ] HTTPS/TLS 已配置
- [ ] SSRF 防护已启用
- [ ] 代码沙箱已配置
- [ ] 数据库备份策略已实施
- [ ] 监控和告警已配置
- [ ] 日志聚合已配置
- [ ] 资源限制已设置
- [ ] 健康检查已配置
- [ ] 文档已更新

### 运行时检查

- [ ] 所有服务健康状态正常
- [ ] 无异常日志
- [ ] 资源使用在合理范围
- [ ] 队列深度正常
- [ ] 响应时间正常
- [ ] 错误率低于阈值
- [ ] 备份正常执行
- [ ] 监控数据正常上报
- [ ] SSL 证书有效
- [ ] 安全扫描无高危漏洞

### 定期检查 (每月)

- [ ] 审查访问日志
- [ ] 检查资源使用趋势
- [ ] 测试备份恢复
- [ ] 更新依赖版本
- [ ] 审查安全配置
- [ ] 更新文档
- [ ] 容量规划评估
- [ ] 故障演练

---

## 参考资料

- [Docker 最佳实践](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes 最佳实践](https://kubernetes.io/docs/concepts/configuration/overview/)
- [OWASP 安全指南](https://owasp.org/www-project-top-ten/)
- [12-Factor App](https://12factor.net/)

---

**最后更新**: 2026-03-04

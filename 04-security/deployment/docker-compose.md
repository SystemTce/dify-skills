# Docker Compose 部署

## 概述

Docker Compose 是 Dify 推荐的部署方式,提供一键部署、易于管理的容器化解决方案。

## 快速开始

### 最小部署

```bash
# 1. 克隆仓库
git clone https://github.com/langgenius/dify.git
cd dify/docker

# 2. 复制环境变量
cp .env.example .env

# 3. 修改关键配置
vim .env
# 修改: SECRET_KEY, DB_PASSWORD, REDIS_PASSWORD

# 4. 启动服务
docker compose up -d

# 5. 验证部署
docker compose ps
curl http://localhost/health
```

### 服务架构

```
nginx (80/443)
  ├── web (3000) - Next.js 前端
  ├── api (5001) - Flask API
  ├── worker - Celery Worker
  ├── beat - Celery Beat
  ├── plugin_daemon (5002) - 插件管理
  └── sandbox (8194) - 代码执行

基础服务:
  ├── db_postgres (5432) - PostgreSQL
  ├── redis (6379) - Redis
  ├── weaviate (8080) - 向量数据库
  └── ssrf_proxy (3128) - SSRF 防护
```

## 环境变量配置

### 核心配置

```bash
# 应用配置
SECRET_KEY=<strong-random-key>  # ⚠️ 必须修改
INIT_PASSWORD=<admin-password>  # 初始管理员密码
DEPLOY_ENV=PRODUCTION

# 数据库配置
DB_TYPE=postgresql
DB_USERNAME=postgres
DB_PASSWORD=<strong-password>   # ⚠️ 必须修改
DB_HOST=db_postgres
DB_PORT=5432
DB_DATABASE=dify

# Redis 配置
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=<strong-password>  # ⚠️ 必须修改
REDIS_DB=0

# Celery 配置
CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@redis:6379/1
CELERY_BACKEND=redis
```

### 安全配置

```bash
# SSRF 防护
SSRF_PROXY_HTTP_URL=http://ssrf_proxy:3128
SSRF_PROXY_HTTPS_URL=http://ssrf_proxy:3128

# 代码沙箱
CODE_EXECUTION_ENDPOINT=http://sandbox:8194
CODE_EXECUTION_API_KEY=<strong-random-key>  # ⚠️ 必须修改
SANDBOX_API_KEY=<strong-random-key>         # ⚠️ 必须修改

# 文件上传限制
UPLOAD_FILE_SIZE_LIMIT=15  # MB
UPLOAD_FILE_BATCH_LIMIT=5
```

### 监控配置

```bash
# OpenTelemetry
ENABLE_OTEL=true
OTLP_BASE_ENDPOINT=http://otel-collector:4318
OTEL_SAMPLING_RATE=0.1

# Sentry
API_SENTRY_DSN=<your-sentry-dsn>
API_SENTRY_TRACES_SAMPLE_RATE=1.0
WEB_SENTRY_DSN=<your-sentry-dsn>
```

## 生产级配置

### docker-compose.prod.yaml

```yaml
version: '3.8'

services:
  api:
    image: langgenius/dify-api:1.13.0
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
      restart_policy:
        condition: on-failure
        max_attempts: 3
        delay: 5s
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    environment:
      MODE: api
      LOG_LEVEL: INFO
      # ... 其他环境变量

  worker:
    image: langgenius/dify-api:1.13.0
    restart: always
    deploy:
      replicas: 2  # 多个 worker 实例
      resources:
        limits:
          cpus: '4'
          memory: 8G
    environment:
      MODE: worker
      CELERY_WORKER_AMOUNT: 4
      # ... 其他环境变量

  nginx:
    image: nginx:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/ssl:ro
      - ./volumes/certbot/conf/live:/etc/letsencrypt/live:ro
    depends_on:
      - api
      - web
```

### 健康检查配置

```yaml
healthcheck:
  # PostgreSQL
  db_postgres:
    test: ["CMD", "pg_isready", "-h", "db_postgres", "-U", "postgres"]
    interval: 1s
    timeout: 3s
    retries: 60

  # Redis
  redis:
    test: ["CMD-SHELL", "redis-cli -a ${REDIS_PASSWORD} ping | grep -q PONG"]
    interval: 5s
    timeout: 3s
    retries: 30

  # API
  api:
    test: ["CMD", "curl", "-f", "http://localhost:5001/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s

  # Sandbox
  sandbox:
    test: ["CMD", "curl", "-f", "http://localhost:8194/health"]
    interval: 30s
    timeout: 10s
    retries: 3
```

### 依赖管理

```yaml
services:
  api:
    depends_on:
      init_permissions:
        condition: service_completed_successfully
      db_postgres:
        condition: service_healthy
      redis:
        condition: service_started
    # ... 其他配置

  worker:
    depends_on:
      db_postgres:
        condition: service_healthy
      redis:
        condition: service_started
    # ... 其他配置
```

## 网络配置

### 网络隔离

```yaml
networks:
  # 内部网络 - 无法访问外部
  ssrf_proxy_network:
    driver: bridge
    internal: true

  # 默认网络 - 可访问外部
  default:
    driver: bridge

services:
  api:
    networks:
      - ssrf_proxy_network
      - default

  sandbox:
    networks:
      - ssrf_proxy_network

  ssrf_proxy:
    networks:
      - ssrf_proxy_network
      - default
```

### 端口映射

```yaml
services:
  nginx:
    ports:
      - "${EXPOSE_NGINX_PORT:-80}:80"
      - "${EXPOSE_NGINX_SSL_PORT:-443}:443"

  plugin_daemon:
    ports:
      - "${EXPOSE_PLUGIN_DEBUGGING_PORT:-5003}:5003"
```

## 存储配置

### 数据持久化

```yaml
services:
  db_postgres:
    volumes:
      - ./volumes/db/data:/var/lib/postgresql/data

  redis:
    volumes:
      - ./volumes/redis/data:/data

  api:
    volumes:
      - ./volumes/app/storage:/app/api/storage

  weaviate:
    volumes:
      - ./volumes/weaviate:/var/lib/weaviate
```

### 备份策略

```bash
# 数据库备份
docker compose exec db_postgres pg_dump -U postgres dify > backup_$(date +%Y%m%d).sql

# 文件存储备份
tar -czf storage_backup_$(date +%Y%m%d).tar.gz ./volumes/app/storage

# 自动备份脚本
#!/bin/bash
BACKUP_DIR=/backups
DATE=$(date +%Y%m%d_%H%M%S)

# 备份数据库
docker compose exec -T db_postgres pg_dump -U postgres dify | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# 备份文件
tar -czf $BACKUP_DIR/storage_$DATE.tar.gz ./volumes/app/storage

# 保留最近 7 天的备份
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete
```

## 日志管理

### 日志配置

```yaml
services:
  api:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "service=api"

  worker:
    logging:
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "5"
```

### 日志查看

```bash
# 查看所有服务日志
docker compose logs

# 查看特定服务日志
docker compose logs api
docker compose logs worker

# 实时查看日志
docker compose logs -f api

# 查看最近 100 行
docker compose logs --tail=100 api

# 查看特定时间范围
docker compose logs --since 2024-01-01T00:00:00 api
```

### 日志聚合

使用 Fluentd 或 Loki 聚合日志:

```yaml
# docker-compose.logging.yaml
services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yaml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

## 常见操作

### 启动和停止

```bash
# 启动所有服务
docker compose up -d

# 启动特定服务
docker compose up -d api worker

# 停止所有服务
docker compose down

# 停止并删除数据
docker compose down -v

# 重启服务
docker compose restart api

# 重新构建并启动
docker compose up -d --build
```

### 扩展服务

```bash
# 扩展 worker 实例
docker compose up -d --scale worker=3

# 查看服务状态
docker compose ps

# 查看资源使用
docker stats
```

### 更新服务

```bash
# 拉取最新镜像
docker compose pull

# 重新创建容器
docker compose up -d --force-recreate

# 滚动更新
docker compose up -d --no-deps --build api
```

## 故障排查

### 容器启动失败

```bash
# 查看容器状态
docker compose ps

# 查看日志
docker compose logs api

# 检查配置
docker compose config

# 验证环境变量
docker compose exec api env | grep DB_
```

### 网络问题

```bash
# 查看网络
docker network ls

# 检查网络连接
docker compose exec api ping db_postgres
docker compose exec api curl http://redis:6379

# 测试 SSRF Proxy
docker compose exec api curl -x http://ssrf_proxy:3128 https://www.google.com
```

### 性能问题

```bash
# 查看资源使用
docker stats

# 查看容器详情
docker compose exec api top

# 检查数据库连接
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT count(*) FROM pg_stat_activity;"
```

## 参考资料

- [Docker Compose 官方文档](https://docs.docker.com/compose/)
- [Dify Docker 部署指南](https://docs.dify.ai/getting-started/install-self-hosted/docker-compose)
- [Docker 最佳实践](https://docs.docker.com/develop/dev-best-practices/)

---

**最后更新**: 2026-03-04

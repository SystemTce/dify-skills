# 故障排查手册

## 概述

本文档提供 Dify 部署和运维中常见问题的诊断和解决方案。

## 问题分类索引

### 部署问题
- [容器启动失败](#容器启动失败)
- [依赖服务未就绪](#依赖服务未就绪)
- [环境变量配置错误](#环境变量配置错误)

### 网络问题
- [无法访问外部 API](#无法访问外部-api)
- [内部服务通信失败](#内部服务通信失败)
- [SSRF Proxy 问题](#ssrf-proxy-问题)

### 数据库问题
- [数据库连接失败](#数据库连接失败)
- [连接池耗尽](#连接池耗尽)
- [慢查询问题](#慢查询问题)

### 性能问题
- [响应慢](#响应慢)
- [内存溢出](#内存溢出)
- [CPU 使用率高](#cpu-使用率高)

### 队列问题
- [任务积压](#任务积压)
- [Worker 无响应](#worker-无响应)
- [任务执行失败](#任务执行失败)

---

## 部署问题

### 容器启动失败

#### 症状
- 容器反复重启
- `docker compose ps` 显示 `Restarting` 或 `Exited`
- 服务无法访问

#### 诊断步骤

```bash
# 1. 查看容器状态
docker compose ps

# 2. 查看日志
docker compose logs api
docker compose logs --tail=100 api

# 3. 检查健康状态
docker compose exec api curl http://localhost:5001/health

# 4. 验证环境变量
docker compose exec api env | grep -E "DB_|REDIS_|SECRET"
```

#### 常见原因和解决方案

**原因 1: 数据库连接失败**

```bash
# 检查数据库
docker compose exec db_postgres pg_isready -h db_postgres -U postgres

# 验证密码
docker compose exec api env | grep DB_PASSWORD

# 测试连接
docker compose exec api python -c "
import psycopg2
conn = psycopg2.connect(
    host='db_postgres',
    database='dify',
    user='postgres',
    password='your-password'
)
print('Connected successfully')
"
```

**解决方案**:
- 确保 `DB_PASSWORD` 与数据库密码一致
- 检查 `depends_on` 配置确保数据库先启动
- 增加健康检查重试次数

**原因 2: SECRET_KEY 未设置**

```bash
# 检查 SECRET_KEY
docker compose exec api env | grep SECRET_KEY

# 生成新密钥
openssl rand -base64 42
```

**解决方案**:
- 在 `.env` 中设置 `SECRET_KEY`
- 重启服务: `docker compose restart api`

**原因 3: 端口冲突**

```bash
# 检查端口占用
netstat -tulpn | grep 5001
lsof -i :5001
```

**解决方案**:
- 修改 `.env` 中的端口配置
- 停止占用端口的进程

### 依赖服务未就绪

#### 症状
- API 启动但立即退出
- 日志显示 "Connection refused"

#### 诊断步骤

```bash
# 检查依赖服务状态
docker compose ps db_postgres redis

# 查看健康检查
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Health}}"

# 测试连接
docker compose exec api ping db_postgres
docker compose exec api curl http://redis:6379
```

#### 解决方案

```yaml
# docker-compose.yaml - 配置正确的依赖
services:
  api:
    depends_on:
      db_postgres:
        condition: service_healthy  # 等待健康检查通过
      redis:
        condition: service_started  # 等待服务启动
```

### 环境变量配置错误

#### 症状
- 服务启动但功能异常
- 日志显示配置错误

#### 诊断步骤

```bash
# 验证环境变量
docker compose config | grep -A 10 "environment:"

# 检查特定变量
docker compose exec api env | grep DB_
docker compose exec api env | grep REDIS_
docker compose exec api env | grep SECRET_KEY
```

#### 常见错误

```bash
# 错误 1: 密码包含特殊字符未转义
DB_PASSWORD=pass@word  # ❌ 错误
DB_PASSWORD='pass@word'  # ✅ 正确

# 错误 2: 布尔值格式错误
DEBUG=True  # ❌ 错误
DEBUG=true  # ✅ 正确

# 错误 3: 数字格式错误
REDIS_PORT="6379"  # ❌ 错误
REDIS_PORT=6379  # ✅ 正确
```

---

## 网络问题

### 无法访问外部 API

#### 症状
- HTTP 请求节点失败
- 日志显示 "Connection timeout" 或 "403 Forbidden"

#### 诊断步骤

```bash
# 1. 测试 SSRF Proxy
docker compose exec api curl -x http://ssrf_proxy:3128 https://www.google.com

# 2. 检查代理配置
docker compose exec api env | grep PROXY

# 3. 查看代理日志
docker compose logs ssrf_proxy | grep "TCP_DENIED"

# 4. 测试 DNS 解析
docker compose exec api nslookup api.openai.com
```

#### 解决方案

**问题 1: SSRF Proxy 阻止访问**

```conf
# squid.conf - 添加允许的域名
acl allowed_domains dstdomain .openai.com .anthropic.com
http_access allow allowed_domains

# 或允许所有外部访问
acl localnet src 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
http_access deny localnet
http_access allow all
```

**问题 2: 端口限制**

```conf
# squid.conf - 允许更多端口
acl Safe_ports port 80 443 8080 8443
http_access deny !Safe_ports
```

**问题 3: 超时设置过短**

```conf
# squid.conf - 增加超时时间
connect_timeout 30 seconds
request_timeout 2 minutes
read_timeout 2 minutes
```

### 内部服务通信失败

#### 症状
- API 无法连接数据库或 Redis
- 日志显示 "Name or service not known"

#### 诊断步骤

```bash
# 1. 检查网络
docker network ls
docker network inspect dify_default

# 2. 测试连接
docker compose exec api ping db_postgres
docker compose exec api ping redis

# 3. 检查 DNS 解析
docker compose exec api nslookup db_postgres
docker compose exec api cat /etc/hosts
```

#### 解决方案

```bash
# 重建网络
docker compose down
docker network prune
docker compose up -d

# 或指定网络
docker compose --project-name dify up -d
```

### SSRF Proxy 问题

#### 症状
- 所有外部请求失败
- 代理日志显示大量错误

#### 诊断步骤

```bash
# 1. 检查代理状态
docker compose ps ssrf_proxy

# 2. 查看日志
docker compose logs ssrf_proxy

# 3. 测试代理
docker compose exec ssrf_proxy curl http://localhost:3128

# 4. 检查配置
docker compose exec ssrf_proxy cat /etc/squid/squid.conf
```

#### 解决方案

```bash
# 重启代理
docker compose restart ssrf_proxy

# 清理缓存
docker compose exec ssrf_proxy rm -rf /var/spool/squid/*
docker compose restart ssrf_proxy

# 验证配置
docker compose exec ssrf_proxy squid -k parse
```

---

## 数据库问题

### 数据库连接失败

#### 症状
- API 启动失败
- 日志显示 "could not connect to server"

#### 诊断步骤

```bash
# 1. 检查数据库状态
docker compose ps db_postgres

# 2. 测试连接
docker compose exec db_postgres pg_isready -h db_postgres -U postgres

# 3. 查看日志
docker compose logs db_postgres

# 4. 验证密码
docker compose exec api env | grep DB_PASSWORD
```

#### 解决方案

```bash
# 重启数据库
docker compose restart db_postgres

# 等待健康检查
docker compose ps db_postgres

# 验证连接
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT 1;"
```

### 连接池耗尽

#### 症状
- 请求变慢或超时
- 日志显示 "connection pool exhausted"

#### 诊断步骤

```sql
-- 查看当前连接数
SELECT count(*) FROM pg_stat_activity;

-- 查看连接详情
SELECT
    datname,
    usename,
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY datname, usename, application_name, state;

-- 查看空闲连接
SELECT count(*) FROM pg_stat_activity WHERE state = 'idle';
```

#### 解决方案

```bash
# .env - 增加连接池大小
SQLALCHEMY_POOL_SIZE=50
SQLALCHEMY_MAX_OVERFLOW=20

# PostgreSQL - 增加最大连接数
POSTGRES_MAX_CONNECTIONS=200

# 重启服务
docker compose restart api worker
```

### 慢查询问题

#### 症状
- 响应时间长
- 数据库 CPU 使用率高

#### 诊断步骤

```sql
-- 启用慢查询日志
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- 1秒
SELECT pg_reload_conf();

-- 查看慢查询
SELECT
    query,
    calls,
    total_time,
    mean_time,
    max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- 查看当前运行的查询
SELECT
    pid,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

#### 解决方案

```sql
-- 添加索引
CREATE INDEX idx_accounts_email ON accounts(email);
CREATE INDEX idx_workflows_tenant_id ON workflows(tenant_id);

-- 分析表
ANALYZE accounts;
ANALYZE workflows;

-- 清理死元组
VACUUM ANALYZE;
```

---

## 性能问题

### 响应慢

#### 症状
- API 响应时间长
- 用户体验差

#### 诊断步骤

```bash
# 1. 查看资源使用
docker stats

# 2. 检查数据库连接
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT count(*) FROM pg_stat_activity;"

# 3. 检查 Redis
docker compose exec redis redis-cli INFO stats

# 4. 查看队列深度
docker compose exec redis redis-cli LLEN celery

# 5. 检查日志
docker compose logs api | grep "slow"
```

#### 解决方案

**优化 1: 增加 Worker 数量**

```bash
# .env
CELERY_WORKER_AMOUNT=4

# 或扩展 Worker 实例
docker compose up -d --scale worker=3
```

**优化 2: 启用缓存**

```bash
# .env
REDIS_DB=0  # 确保 Redis 可用

# 启用 LLM 响应缓存
# 在工作流中配置缓存节点
```

**优化 3: 数据库优化**

```sql
-- 增加共享缓冲区
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '1GB';
SELECT pg_reload_conf();
```

### 内存溢出

#### 症状
- 容器被 OOM Killer 终止
- 日志显示 "Out of memory"

#### 诊断步骤

```bash
# 1. 查看内存使用
docker stats

# 2. 检查容器限制
docker inspect api | grep -A 10 "Memory"

# 3. 查看系统日志
dmesg | grep -i "out of memory"
```

#### 解决方案

```yaml
# docker-compose.yaml - 增加内存限制
services:
  api:
    deploy:
      resources:
        limits:
          memory: 4G
        reservations:
          memory: 2G

  worker:
    deploy:
      resources:
        limits:
          memory: 8G
```

### CPU 使用率高

#### 症状
- 容器 CPU 使用率持续 100%
- 响应变慢

#### 诊断步骤

```bash
# 1. 查看 CPU 使用
docker stats

# 2. 查看进程
docker compose exec api top

# 3. 检查慢查询
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT * FROM pg_stat_activity WHERE state = 'active';"
```

#### 解决方案

```yaml
# docker-compose.yaml - 限制 CPU
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '2'

# 或增加 CPU 资源
# 升级服务器配置
```

---

## 队列问题

### 任务积压

#### 症状
- 队列长度持续增长
- 任务处理延迟

#### 诊断步骤

```bash
# 1. 查看队列长度
docker compose exec redis redis-cli LLEN celery

# 2. 查看 Worker 状态
docker compose logs worker

# 3. 检查任务执行时间
docker compose logs worker | grep "Task.*succeeded"
```

#### 解决方案

```bash
# 增加 Worker 数量
docker compose up -d --scale worker=5

# 或增加 Worker 并发数
# .env
CELERY_WORKER_AMOUNT=8
```

### Worker 无响应

#### 症状
- 任务不执行
- Worker 日志无输出

#### 诊断步骤

```bash
# 1. 检查 Worker 状态
docker compose ps worker

# 2. 查看日志
docker compose logs worker

# 3. 测试 Celery 连接
docker compose exec worker python -c "
from celery import Celery
app = Celery('dify', broker='redis://:password@redis:6379/1')
print(app.control.inspect().active())
"
```

#### 解决方案

```bash
# 重启 Worker
docker compose restart worker

# 清理僵尸任务
docker compose exec redis redis-cli DEL celery

# 检查 Celery 配置
docker compose exec worker env | grep CELERY
```

### 任务执行失败

#### 症状
- 任务状态为 FAILURE
- 日志显示错误

#### 诊断步骤

```bash
# 1. 查看失败任务
docker compose logs worker | grep "FAILURE"

# 2. 检查错误详情
docker compose logs worker | grep -A 10 "Traceback"

# 3. 验证依赖服务
docker compose ps
```

#### 解决方案

```bash
# 根据错误类型解决:

# 1. 数据库连接失败
docker compose restart db_postgres

# 2. 外部 API 超时
# 增加超时时间或重试次数

# 3. 内存不足
# 增加 Worker 内存限制
```

---

## 快速诊断命令

```bash
# 健康检查
curl http://localhost/health
docker compose ps
docker compose exec db_postgres pg_isready
docker compose exec redis redis-cli ping

# 日志查看
docker compose logs -f api
docker compose logs --tail=100 worker
docker compose logs --since 10m api

# 资源监控
docker stats
docker compose exec api top
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT count(*) FROM pg_stat_activity;"

# 网络测试
docker compose exec api ping db_postgres
docker compose exec api curl http://redis:6379
docker compose exec api curl -x http://ssrf_proxy:3128 https://www.google.com

# 数据库诊断
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT * FROM pg_stat_activity;"
docker compose exec db_postgres psql -U postgres -d dify -c "SELECT * FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;"

# Redis 诊断
docker compose exec redis redis-cli INFO
docker compose exec redis redis-cli LLEN celery
docker compose exec redis redis-cli SLOWLOG GET 10
```

## 参考资料

- [Docker 故障排查](https://docs.docker.com/config/containers/logging/)
- [PostgreSQL 性能调优](https://www.postgresql.org/docs/current/performance-tips.html)
- [Redis 故障排查](https://redis.io/docs/management/optimization/)
- [Celery 故障排查](https://docs.celeryproject.org/en/stable/userguide/monitoring.html)

---

**最后更新**: 2026-03-04

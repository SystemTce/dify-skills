# 代码沙箱隔离

## 概述

Dify Sandbox 提供安全的代码执行环境,隔离用户提交的 Python 和 JavaScript 代码,防止恶意代码影响系统安全。

## 架构设计

```
┌──────────────────┐
│  Workflow Node   │
│  (Code Execute)  │
└────────┬─────────┘
         │ gRPC/HTTP
         ↓
┌──────────────────┐
│  Sandbox Service │
│  (Port 8194)     │
├──────────────────┤
│  Worker Pool     │ ← 资源限制
│  (Max 4 workers) │
└────────┬─────────┘
         │ 隔离执行
         ↓
┌──────────────────┐
│  Isolated Env    │
│  - Syscall限制   │
│  - 网络代理      │
│  - 文件系统隔离  │
└──────────────────┘
```

## 核心配置

### config.yaml 配置

```yaml
# 应用配置
app:
  port: 8194
  debug: false
  key: dify-sandbox  # ⚠️ 生产环境必须修改

# Worker 配置
max_workers: 4              # 最大并发 worker 数
max_requests: 50            # 每个 worker 最大请求数
worker_timeout: 15          # Worker 超时时间(秒)

# Python 环境
python_path: /usr/local/bin/python3
python_lib_path:
  - /usr/local/lib/python3.10
  - /usr/lib/python3.10
  - /usr/lib/python3
  - /usr/lib/x86_64-linux-gnu
  # 系统文件
  - /etc/ssl/certs/ca-certificates.crt
  - /etc/nsswitch.conf
  - /etc/hosts
  - /etc/resolv.conf
  - /etc/localtime
  - /usr/share/zoneinfo
  - /etc/timezone

# Python 包镜像
python_pip_mirror_url: https://pypi.tuna.tsinghua.edu.cn/simple

# Node.js 环境
nodejs_path: /usr/local/bin/node

# 网络配置
enable_network: true        # 启用网络访问
proxy:
  socks5: ''
  http: 'http://ssrf_proxy:3128'    # HTTP 代理
  https: 'http://ssrf_proxy:3128'   # HTTPS 代理

# 系统调用白名单
allowed_syscalls:
  - 1    # write
  - 2    # open
  - 3    # close
  # ... 添加需要的系统调用
```

### 环境变量配置

```bash
# .env
SANDBOX_API_KEY=dify-sandbox          # ⚠️ 必须修改
SANDBOX_GIN_MODE=release
SANDBOX_WORKER_TIMEOUT=15
SANDBOX_ENABLE_NETWORK=true
SANDBOX_HTTP_PROXY=http://ssrf_proxy:3128
SANDBOX_HTTPS_PROXY=http://ssrf_proxy:3128
SANDBOX_PORT=8194

# 代码执行限制
CODE_EXECUTION_ENDPOINT=http://sandbox:8194
CODE_EXECUTION_API_KEY=dify-sandbox  # ⚠️ 必须修改
CODE_EXECUTION_CONNECT_TIMEOUT=10
CODE_EXECUTION_READ_TIMEOUT=60
CODE_EXECUTION_WRITE_TIMEOUT=10

# 代码限制
CODE_MAX_NUMBER=9223372036854775807
CODE_MIN_NUMBER=-9223372036854775808
CODE_MAX_DEPTH=5
CODE_MAX_PRECISION=20
CODE_MAX_STRING_LENGTH=400000
CODE_MAX_STRING_ARRAY_LENGTH=30
CODE_MAX_OBJECT_ARRAY_LENGTH=30
CODE_MAX_NUMBER_ARRAY_LENGTH=1000
```

## 安全特性

### 1. 系统调用限制

使用 seccomp 限制允许的系统调用:

```yaml
allowed_syscalls:
  - 1    # write - 写文件
  - 2    # open - 打开文件
  - 3    # close - 关闭文件
  - 4    # stat - 获取文件状态
  - 5    # fstat - 获取文件描述符状态
  - 9    # mmap - 内存映射
  - 10   # mprotect - 内存保护
  - 11   # munmap - 取消内存映射
  - 12   # brk - 改变数据段大小
  - 21   # access - 检查文件权限
  - 59   # execve - 执行程序
  - 231  # exit_group - 退出进程组
```

**被阻止的危险系统调用**:
- `fork` - 创建进程
- `kill` - 终止进程
- `ptrace` - 进程跟踪
- `mount` - 挂载文件系统
- `reboot` - 重启系统

### 2. 资源配额管理

```yaml
# Worker 资源限制
max_workers: 4              # 最大并发数
worker_timeout: 15          # 执行超时
max_requests: 50            # 最大请求数

# 代码执行限制
CODE_MAX_DEPTH: 5           # 最大嵌套深度
CODE_MAX_STRING_LENGTH: 400000  # 最大字符串长度
CODE_MAX_NUMBER_ARRAY_LENGTH: 1000  # 最大数组长度
```

### 3. 网络访问控制

所有网络请求通过 SSRF Proxy:

```yaml
enable_network: true
proxy:
  http: 'http://ssrf_proxy:3128'
  https: 'http://ssrf_proxy:3128'
```

**优势**:
- ✅ 阻止访问内网地址
- ✅ 记录所有外部请求
- ✅ 可配置域名白名单/黑名单

### 4. 文件系统隔离

仅允许访问特定路径:

```yaml
python_lib_path:
  - /usr/local/lib/python3.10  # Python 库
  - /etc/ssl/certs             # SSL 证书
  - /etc/hosts                 # 主机名解析
  # 其他路径被阻止
```

## 适用场景

### 场景 1: 生产环境严格隔离

**需求**: 最大化安全性,最小化权限

**配置**:

```yaml
app:
  debug: false
  key: <strong-random-key>

max_workers: 4
worker_timeout: 10          # 更短的超时
enable_network: true        # 通过代理访问
proxy:
  http: 'http://ssrf_proxy:3128'
  https: 'http://ssrf_proxy:3128'

# 最小系统调用集
allowed_syscalls:
  - 1    # write
  - 2    # open
  - 3    # close
  - 231  # exit_group
```

**适用于**:
- 多租户 SaaS 环境
- 处理不可信代码
- 高安全要求场景

### 场景 2: 开发环境灵活配置

**需求**: 方便调试,支持更多功能

**配置**:

```yaml
app:
  debug: true               # 启用调试
  key: dify-sandbox

max_workers: 2
worker_timeout: 30          # 更长的超时
enable_network: true

# 更宽松的系统调用
allowed_syscalls:
  - 1-100                   # 允许更多系统调用
```

**适用于**:
- 本地开发环境
- 测试环境
- 调试代码执行问题

### 场景 3: 离线环境

**需求**: 无外部网络访问

**配置**:

```yaml
enable_network: false       # 禁用网络
proxy:
  http: ''
  https: ''

# 使用本地 pip 镜像
python_pip_mirror_url: http://local-pypi-mirror/simple
```

**适用于**:
- 内网部署
- 离线环境
- 无需外部 API 调用

## 验证方法

### 测试代码执行

```python
# 测试 Python 代码执行
import requests

response = requests.post(
    'http://localhost:8194/v1/sandbox/run',
    headers={'X-Api-Key': 'dify-sandbox'},
    json={
        'language': 'python3',
        'code': 'print("Hello from sandbox")',
        'preload': ''
    }
)
print(response.json())
```

### 测试网络隔离

```python
# 测试内网访问被阻止
code = '''
import requests
response = requests.get('http://127.0.0.1')
print(response.text)
'''

# 应该失败或返回 403
```

### 测试资源限制

```python
# 测试超时限制
code = '''
import time
time.sleep(20)  # 超过 worker_timeout
'''

# 应该超时并返回错误
```

### 查看 Sandbox 日志

```bash
# 查看日志
docker compose logs -f sandbox

# 查看健康状态
curl http://localhost:8194/health
```

## 注意事项

### ⚠️ API Key 安全

生产环境必须修改默认 API Key:

```bash
# 生成强密钥
SANDBOX_API_KEY=$(openssl rand -base64 32)

# 更新 .env
CODE_EXECUTION_API_KEY=$SANDBOX_API_KEY
```

### ⚠️ Worker 数量配置

根据 CPU 核心数配置:

```yaml
# 2 核 CPU
max_workers: 2

# 4 核 CPU
max_workers: 4

# 8 核 CPU
max_workers: 6-8
```

### ⚠️ 超时时间设置

根据代码复杂度设置:

```yaml
# 简单计算
worker_timeout: 5

# 一般任务
worker_timeout: 15

# 复杂任务
worker_timeout: 30
```

### ⚠️ 网络代理配置

确保代理正常工作:

```bash
# 测试代理连接
docker compose exec sandbox curl -x http://ssrf_proxy:3128 https://www.google.com
```

## 故障排查

### 问题 1: 代码执行超时

**症状**: 代码执行总是超时

**原因**: worker_timeout 设置过短

**解决方案**:

```yaml
# 增加超时时间
worker_timeout: 30

# 或优化代码减少执行时间
```

### 问题 2: 网络请求失败

**症状**: 代码中的网络请求失败

**原因**: 代理配置错误或未启用网络

**解决方案**:

```yaml
# 检查网络配置
enable_network: true
proxy:
  http: 'http://ssrf_proxy:3128'
  https: 'http://ssrf_proxy:3128'

# 验证代理连接
docker compose exec sandbox curl -x http://ssrf_proxy:3128 https://api.openai.com
```

### 问题 3: 系统调用被阻止

**症状**: 代码执行失败,日志显示系统调用被拒绝

**原因**: allowed_syscalls 配置过于严格

**解决方案**:

```yaml
# 添加需要的系统调用
allowed_syscalls:
  - 1-100  # 临时允许更多系统调用进行调试

# 查看日志确定需要的系统调用
docker compose logs sandbox | grep "syscall"
```

### 问题 4: Worker 耗尽

**症状**: 请求排队,响应慢

**原因**: max_workers 设置过小

**解决方案**:

```yaml
# 增加 worker 数量
max_workers: 8

# 或增加 max_requests
max_requests: 100
```

## 高级配置

### 自定义 Python 包

```yaml
# 预装 Python 包
python_lib_path:
  - /usr/local/lib/python3.10
  - /custom/python/packages  # 自定义包路径
```

```bash
# 安装自定义包
docker compose exec sandbox pip install -t /custom/python/packages requests pandas
```

### 多语言支持

```yaml
# Python 配置
python_path: /usr/local/bin/python3

# Node.js 配置
nodejs_path: /usr/local/bin/node

# 其他语言可通过扩展支持
```

### 资源监控

```bash
# 监控 Worker 状态
curl http://localhost:8194/metrics

# 查看资源使用
docker stats sandbox
```

## 性能优化

### Worker 池优化

```yaml
# 根据负载调整
max_workers: 4              # 并发数
max_requests: 50            # 每个 worker 处理请求数
worker_timeout: 15          # 超时时间

# 高负载场景
max_workers: 8
max_requests: 100
```

### 网络优化

```yaml
# 使用本地 pip 镜像
python_pip_mirror_url: https://pypi.tuna.tsinghua.edu.cn/simple

# 配置代理超时
proxy:
  http: 'http://ssrf_proxy:3128'
  https: 'http://ssrf_proxy:3128'
  timeout: 30
```

### 内存优化

```yaml
# 限制字符串和数组大小
CODE_MAX_STRING_LENGTH: 400000
CODE_MAX_NUMBER_ARRAY_LENGTH: 1000
CODE_MAX_OBJECT_ARRAY_LENGTH: 30
```

## 监控和审计

### 健康检查

```bash
# 检查服务健康
curl http://localhost:8194/health

# 检查 Worker 状态
curl http://localhost:8194/workers
```

### 日志分析

```bash
# 查看执行日志
docker compose logs sandbox | grep "execute"

# 查看错误日志
docker compose logs sandbox | grep "error"

# 统计执行次数
docker compose logs sandbox | grep "execute" | wc -l
```

### 性能监控

```bash
# 监控资源使用
docker stats sandbox

# 查看 Worker 队列
curl http://localhost:8194/metrics | grep "queue"
```

## 参考资料

- [Dify Sandbox GitHub](https://github.com/langgenius/dify-sandbox)
- [Seccomp 文档](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html)
- [Docker 安全最佳实践](https://docs.docker.com/engine/security/)

---

**最后更新**: 2026-03-04

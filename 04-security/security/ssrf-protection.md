# SSRF 防护配置

## 概述

SSRF (Server-Side Request Forgery) 是一种安全漏洞,攻击者可以利用服务器发起请求访问内网资源。Dify 使用 Squid 代理实现 SSRF 防护,阻止访问内网地址和敏感端口。

## 架构设计

```
┌─────────────┐
│   API/Worker│
└──────┬──────┘
       │ HTTP Request
       ↓
┌─────────────┐
│ SSRF Proxy  │ ← ACL 规则过滤
│  (Squid)    │
└──────┬──────┘
       │ 允许的请求
       ↓
┌─────────────┐
│ 外部 API    │
└─────────────┘

内网地址 (10.0.0.0/8, 192.168.0.0/16) → 被阻止
```

## 核心配置

### squid.conf 配置

```conf
# 定义内网地址段
acl localnet src 0.0.0.1-0.255.255.255    # RFC 1122 "this" network
acl localnet src 10.0.0.0/8               # RFC 1918 private network
acl localnet src 100.64.0.0/10            # RFC 6598 shared address
acl localnet src 169.254.0.0/16           # RFC 3927 link-local
acl localnet src 172.16.0.0/12            # RFC 1918 private network
acl localnet src 192.168.0.0/16           # RFC 1918 private network
acl localnet src fc00::/7                 # RFC 4193 local private network
acl localnet src fe80::/10                # RFC 4291 link-local

# 定义安全端口
acl SSL_ports port 443
acl Safe_ports port 80                    # http
acl Safe_ports port 443                   # https
acl Safe_ports port 1025-65535            # unregistered ports

# 定义允许的域名 (可选)
acl allowed_domains dstdomain .marketplace.dify.ai

# 访问控制规则
http_access allow allowed_domains         # 允许特定域名
http_access deny !Safe_ports              # 拒绝不安全端口
http_access deny CONNECT !SSL_ports       # 拒绝非 SSL CONNECT
http_access deny localnet                 # 拒绝内网访问
http_access allow localhost               # 允许本地访问
http_access deny all                      # 默认拒绝

# 代理端口
http_port 3128

# 性能优化
connect_timeout 30 seconds
request_timeout 2 minutes
read_timeout 2 minutes
client_lifetime 5 minutes

# 日志配置
logformat dify_log %ts.%03tu %6tr %>a %Ss/%03>Hs %<st %rm %ru
access_log daemon:/var/log/squid/access.log dify_log
```

## 适用场景

### 场景 1: 生产环境严格防护

**需求**: 完全阻止内网访问,仅允许特定外部 API

**配置**:

```conf
# 白名单模式 - 仅允许特定域名
acl allowed_apis dstdomain .openai.com .anthropic.com .google.com
http_access allow allowed_apis
http_access deny all
```

**适用于**:
- 多租户 SaaS 环境
- 高安全要求场景
- 需要审计所有外部访问

### 场景 2: 开发环境灵活配置

**需求**: 允许访问本地服务,但仍阻止敏感内网段

**配置**:

```conf
# 允许本地开发服务
acl dev_services dst 127.0.0.1 localhost

# 仍然阻止内网
acl localnet src 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
http_access deny localnet

# 允许开发服务
http_access allow dev_services
```

**适用于**:
- 本地开发环境
- 测试环境
- 需要访问本地 Mock 服务

### 场景 3: 企业内网部署

**需求**: 允许访问企业内部 API,但阻止其他内网

**配置**:

```conf
# 允许企业内部 API
acl internal_apis dst 10.100.0.0/16
http_access allow internal_apis

# 阻止其他内网
acl other_localnet src 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
http_access deny other_localnet
```

**适用于**:
- 企业私有部署
- 需要访问内部服务
- 混合云环境

## 验证方法

### 测试内网访问被阻止

```bash
# 测试访问 127.0.0.1
docker compose exec api curl -x http://ssrf_proxy:3128 http://127.0.0.1
# 预期: HTTP 403 Forbidden

# 测试访问 10.0.0.1
docker compose exec api curl -x http://ssrf_proxy:3128 http://10.0.0.1
# 预期: HTTP 403 Forbidden

# 测试访问 192.168.1.1
docker compose exec api curl -x http://ssrf_proxy:3128 http://192.168.1.1
# 预期: HTTP 403 Forbidden
```

### 测试外部访问正常

```bash
# 测试访问 OpenAI API
docker compose exec api curl -x http://ssrf_proxy:3128 https://api.openai.com
# 预期: 正常返回或 401 (需要 API Key)

# 测试访问 Google
docker compose exec api curl -x http://ssrf_proxy:3128 https://www.google.com
# 预期: 正常返回 HTML
```

### 查看代理日志

```bash
# 查看访问日志
docker compose exec ssrf_proxy tail -f /var/log/squid/access.log

# 查看被拒绝的请求
docker compose exec ssrf_proxy grep "TCP_DENIED" /var/log/squid/access.log
```

## 注意事项

### ⚠️ 内网地址段完整性

确保覆盖所有内网地址段:

```conf
# IPv4 私有地址
10.0.0.0/8          # Class A
172.16.0.0/12       # Class B
192.168.0.0/16      # Class C

# 特殊地址
127.0.0.0/8         # Loopback
169.254.0.0/16      # Link-local
0.0.0.0/8           # Current network

# IPv6 私有地址
fc00::/7            # Unique local
fe80::/10           # Link-local
```

### ⚠️ DNS 解析安全

防止 DNS 重绑定攻击:

```conf
# 禁用 DNS 缓存或使用安全的 DNS 服务器
dns_nameservers 8.8.8.8 8.8.4.4
dns_timeout 30 seconds
```

### ⚠️ 端口限制

限制允许的端口范围:

```conf
# 仅允许常用端口
acl Safe_ports port 80 443
acl Safe_ports port 8080 8443  # 如需支持非标准端口

# 拒绝其他端口
http_access deny !Safe_ports
```

### ⚠️ 性能影响

代理会增加请求延迟:

```conf
# 优化性能
connect_timeout 10 seconds      # 连接超时
request_timeout 60 seconds      # 请求超时
persistent_request_timeout 30 seconds  # 持久连接超时

# 连接池
server_persistent_connections on
client_persistent_connections on
```

## 故障排查

### 问题 1: 外部 API 无法访问

**症状**: 所有外部请求返回 403

**原因**: Safe_ports 配置过于严格

**解决方案**:

```conf
# 检查端口配置
acl Safe_ports port 80 443 1025-65535

# 或添加特定端口
acl Safe_ports port 8080 8443
```

### 问题 2: 内网仍可访问

**症状**: 内网地址可以访问

**原因**: ACL 规则顺序错误

**解决方案**:

```conf
# 确保 deny 规则在 allow 之前
http_access deny localnet
http_access allow localhost
http_access deny all
```

### 问题 3: 代理性能差

**症状**: 请求延迟高,超时频繁

**原因**: 连接池配置不当

**解决方案**:

```conf
# 增加文件描述符限制
max_filedescriptors 65536

# 优化连接池
server_idle_pconn_timeout 2 minutes
client_idle_pconn_timeout 2 minutes

# 增加内存缓存
cache_mem 256 MB
```

### 问题 4: 日志过大

**症状**: 日志文件占用大量磁盘空间

**原因**: 日志轮转配置不当

**解决方案**:

```conf
# 配置日志轮转
logfile_rotate 10
access_log daemon:/var/log/squid/access.log dify_log rotate=10

# 或使用 logrotate
# /etc/logrotate.d/squid
/var/log/squid/*.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
    create 640 proxy proxy
}
```

## 高级配置

### 白名单模式

仅允许特定域名访问:

```conf
# 定义允许的域名
acl allowed_domains dstdomain .openai.com
acl allowed_domains dstdomain .anthropic.com
acl allowed_domains dstdomain .google.com

# 仅允许白名单
http_access allow allowed_domains
http_access deny all
```

### 黑名单模式

阻止特定域名:

```conf
# 定义禁止的域名
acl blocked_domains dstdomain .example-malicious.com
acl blocked_domains dstdomain .spam-site.com

# 拒绝黑名单
http_access deny blocked_domains
```

### 基于时间的访问控制

```conf
# 定义时间段
acl business_hours time MTWHF 09:00-18:00

# 仅在工作时间允许访问
http_access allow business_hours
http_access deny all
```

### 带宽限制

```conf
# 限制下载速度
delay_pools 1
delay_class 1 1
delay_parameters 1 1000000/1000000  # 1 MB/s
delay_access 1 allow all
```

## 监控和审计

### 访问日志分析

```bash
# 统计访问最多的域名
docker compose exec ssrf_proxy awk '{print $7}' /var/log/squid/access.log | sort | uniq -c | sort -rn | head -10

# 统计被拒绝的请求
docker compose exec ssrf_proxy grep "TCP_DENIED" /var/log/squid/access.log | wc -l

# 查看特定 IP 的请求
docker compose exec ssrf_proxy grep "10.0.0.1" /var/log/squid/access.log
```

### 实时监控

```bash
# 实时查看访问日志
docker compose exec ssrf_proxy tail -f /var/log/squid/access.log

# 实时查看被拒绝的请求
docker compose exec ssrf_proxy tail -f /var/log/squid/access.log | grep "TCP_DENIED"
```

### 告警配置

```bash
# 监控脚本示例
#!/bin/bash
THRESHOLD=100
DENIED_COUNT=$(docker compose exec ssrf_proxy grep "TCP_DENIED" /var/log/squid/access.log | wc -l)

if [ $DENIED_COUNT -gt $THRESHOLD ]; then
    echo "Alert: High number of denied requests: $DENIED_COUNT"
    # 发送告警邮件或通知
fi
```

## 参考资料

- [Squid 官方文档](http://www.squid-cache.org/Doc/)
- [OWASP SSRF 防护](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [Dify SSRF 防护说明](https://docs.dify.ai/learn-more/faq/install-faq#18-why-is-ssrf-proxy-needed)

---

**最后更新**: 2026-03-04

# 常见问题排查

## 插件启动失败

### 问题：插件无法启动

**可能原因**：

1. **Python 版本不匹配**
   ```bash
   # 检查 Python 版本
   python --version
   # 需要 >= 3.12
   ```

2. **依赖安装失败**
   ```bash
   # 重新安装依赖
   uv pip install -r requirements.txt
   ```

3. **manifest.yaml 配置错误**
   ```bash
   # 验证配置
   dify plugin validate
   ```

4. **入口文件路径错误**
   - 检查 `manifest.yaml` 中的 `entrypoint`
   - 确保 `main.py` 文件存在

### 解决方案

```bash
# 1. 检查 Python 版本
python --version

# 2. 重新安装依赖
rm -rf .venv
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt

# 3. 验证配置
dify plugin validate

# 4. 查看详细日志
dify plugin run --debug
```

## 权限验证失败

### 问题：credentials 验证失败

**可能原因**：

1. **API Key 无效**
2. **OAuth 令牌过期**
3. **权限配置错误**
4. **网络连接问题**

### 解决方案

```python
# 在 _validate_credentials 中添加详细日志
def _validate_credentials(self, credentials: dict[str, Any]) -> None:
    api_key = credentials.get("api_key")
    print(f"Validating API Key: {api_key[:10]}...")  # 只打印前10个字符

    try:
        response = requests.get(
            "https://api.example.com/test",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=10
        )
        print(f"Response status: {response.status_code}")
        response.raise_for_status()
    except Exception as e:
        print(f"Validation error: {str(e)}")
        raise ToolProviderCredentialValidationError(str(e))
```

## 运行时错误

### 问题：插件执行超时

**可能原因**：
- 默认超时时间太短
- API 响应慢
- 网络延迟

**解决方案**：

```python
# 在 main.py 中增加超时时间
from dify_plugin import Plugin, DifyPluginEnv

plugin = Plugin(DifyPluginEnv(MAX_REQUEST_TIMEOUT=300))  # 5 分钟

if __name__ == '__main__':
    plugin.run()
```

### 问题：内存溢出

**可能原因**：
- 处理大文件
- 内存限制太小
- 没有使用流式处理

**解决方案**：

```yaml
# 在 manifest.yaml 中增加内存限制
resource:
  memory: 536870912  # 512MB
```

```python
# 使用流式处理
def _invoke(self, tool_parameters):
    # 不要一次性加载所有数据
    for chunk in process_data_in_chunks():
        yield self.create_text_message(chunk)
```

### 问题：网络连接失败

**可能原因**：
- 防火墙阻止
- DNS 解析失败
- 代理配置问题

**解决方案**：

```python
# 添加重试和超时
import requests
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def _call_api(self, url, data):
    response = requests.post(
        url,
        json=data,
        timeout=30,  # 30 秒超时
        proxies={"http": None, "https": None}  # 禁用代理
    )
    response.raise_for_status()
    return response.json()
```

## 调试技巧

### 1. 使用 Debug Runtime

```bash
# 启动 Debug Runtime
dify plugin run --debug

# 查看实时日志
tail -f ~/.dify/logs/plugin.log
```

### 2. 添加日志

```python
import logging

logger = logging.getLogger(__name__)

class MyTool(Tool):
    def _invoke(self, tool_parameters):
        logger.info(f"Tool invoked with parameters: {tool_parameters}")

        try:
            result = self._process(tool_parameters)
            logger.info(f"Tool completed successfully")
            yield self.create_text_message(result)
        except Exception as e:
            logger.error(f"Tool failed: {str(e)}", exc_info=True)
            yield self.create_text_message(f"Error: {str(e)}")
```

### 3. 使用 pytest 调试

```python
# tests/test_my_tool.py
import pytest
from tools.my_tool import MyTool

def test_my_tool():
    tool = MyTool()

    # 设置断点
    import pdb; pdb.set_trace()

    result = list(tool.invoke(tool_parameters={"text": "test"}))
    assert len(result) == 1
```

```bash
# 运行测试并进入调试器
pytest tests/test_my_tool.py -s
```

### 4. 检查 OpenTelemetry 追踪

```bash
# 查看追踪日志
grep "trace_id" ~/.dify/logs/plugin.log
```

## 常见错误代码

| 错误代码 | 含义 | 解决方案 |
|---------|------|---------|
| 400 | 请求参数错误 | 检查参数格式和类型 |
| 401 | 认证失败 | 检查 API Key 或 OAuth 令牌 |
| 403 | 权限不足 | 检查权限配置 |
| 404 | 资源不存在 | 检查 API 端点 |
| 429 | 速率限制 | 添加重试机制 |
| 500 | 服务器错误 | 联系服务提供商 |
| 502 | 网关错误 | 检查网络连接 |
| 504 | 网关超时 | 增加超时时间 |

## 性能问题

### 问题：插件响应慢

**排查步骤**：

1. **测量 API 响应时间**
   ```python
   import time

   start = time.time()
   response = requests.post(url, json=data)
   duration = time.time() - start
   print(f"API call took {duration:.2f} seconds")
   ```

2. **检查网络延迟**
   ```bash
   ping api.example.com
   traceroute api.example.com
   ```

3. **使用缓存**
   ```python
   from functools import lru_cache

   @lru_cache(maxsize=100)
   def get_cached_data(key):
       return expensive_operation(key)
   ```

4. **使用异步处理**
   ```python
   import asyncio
   import aiohttp

   async def _call_api_async(self, url, data):
       async with aiohttp.ClientSession() as session:
           async with session.post(url, json=data) as response:
               return await response.json()
   ```

## 获取帮助

### 官方资源

- [Dify Discord](https://discord.gg/FngNHpbcY7)
- [Dify Reddit](https://reddit.com/r/difyai)
- [GitHub Issues](https://github.com/langgenius/dify/issues)

### 提问模板

```markdown
**问题描述**：
简要描述问题

**环境信息**：
- Dify 版本：
- Python 版本：
- 操作系统：
- 插件类型：

**重现步骤**：
1. 步骤 1
2. 步骤 2
3. 步骤 3

**错误日志**：
```
粘贴错误日志
```

**已尝试的解决方案**：
- 尝试 1
- 尝试 2
```

## 预防措施

1. **参数验证**：在处理前验证所有参数
2. **错误处理**：捕获所有可能的异常
3. **超时设置**：设置合理的超时时间
4. **重试机制**：对临时错误实现重试
5. **日志记录**：记录关键操作和错误
6. **单元测试**：编写完整的测试用例
7. **集成测试**：在真实环境中测试

# CLI 命令参考

> Dify 命令行工具完整命令参考

## 安装

```bash
pip install dify-cli
```

## 初始化配置

```bash
# 设置默认 API Key
dify config set api-key YOUR_API_KEY

# 设置默认基础 URL
dify config set base-url https://api.dify.ai/v1

# 查看配置
dify config list
```

## 插件命令

### 初始化插件

```bash
dify plugin init [NAME] [OPTIONS]

选项:
  --template TEXT    插件模板 (tool, model, datasource)
  --directory PATH   创建目录
  --yes, -y          跳过确认
```

**示例**:

```bash
dify plugin init my-tool --template tool
```

### 运行插件

```bash
dify plugin run [PATH] [OPTIONS]

选项:
  --port NUM        端口号 (默认: 5000)
  --host TEXT       主机地址 (默认: localhost)
  --debug           调试模式
  --reload          热重载
```

**示例**:

```bash
dify plugin run ./my-plugin --port 5000 --debug
```

### 打包插件

```bash
dify plugin package [PATH] [OPTIONS]

选项:
  --output PATH, -o PATH  输出目录
  --format FORMAT         打包格式 (zip, tar.gz)
```

**示例**:

```bash
dify plugin package ./my-plugin -o ./dist
```

### 发布插件

```bash
dify plugin publish [PATH] [OPTIONS]

选项:
  --token TEXT    市场 API Token
  --public       发布到公共市场
```

### 列出已安装插件

```bash
dify plugin list [OPTIONS]

选项:
  --verbose, -v  详细输出
```

## 工作流命令

### 导出工作流

```bash
dify workflow export WORKFLOW_ID [OPTIONS]

选项:
  --output PATH, -o PATH  输出文件路径
  --format FORMAT        格式 (yaml, json)
  --api-key TEXT        API Key
```

**示例**:

```bash
dify workflow export wf_xxx -o workflow.yaml
```

### 导入工作流

```bash
dify workflow import FILE [OPTIONS]

选项:
  --api-key TEXT    API Key
  --validate       验证后导入
```

### 验证工作流

```bash
dify workflow validate FILE

示例:
  dify workflow validate workflow.yaml
```

### 运行工作流

```bash
dify workflow run WORKFLOW_ID [OPTIONS]

选项:
  --inputs TEXT, -i TEXT   输入参数 (JSON 格式)
  --user TEXT              用户标识
  --watch                 实时输出
```

**示例**:

```bash
dify workflow run wf_xxx -i '{"query": "测试"}' --user user123
```

## 应用命令

### 列出应用

```bash
dify app list [OPTIONS]

选项:
  --page NUM     页码
  --limit NUM    每页数量
  --api-key TEXT API Key
```

### 获取应用信息

```bash
dify app info APP_ID [OPTIONS]
```

### 部署应用

```bash
dify app deploy APP_ID [OPTIONS]

选项:
  --environment TEXT  环境 (production, staging)
  --version TEXT     版本
```

## 数据集命令

### 列出数据集

```bash
dify dataset list [OPTIONS]

选项:
  --page NUM
  --limit NUM
```

### 创建数据集

```bash
dify dataset create NAME [OPTIONS]

选项:
  --description TEXT     描述
  --indexing-technique TEXT  索引技术 (high_quality, economy)
```

### 导入文档

```bash
dify dataset import DATASET_ID FILE [OPTIONS]

选项:
  --indexing-technique TEXT  索引技术
  --batch-size NUM         批处理大小
```

## 环境变量命令

### 列出环境变量

```bash
dify env list
```

### 设置环境变量

```bash
dify env set KEY VALUE

示例:
  dify env set API_KEY xxx
```

### 获取环境变量

```bash
dify env get KEY
```

### 删除环境变量

```bash
dify env delete KEY
```

## 调试和日志

### 调试模式

```bash
# 启用调试输出
dify --debug [COMMAND]

# 查看日志
dify logs [OPTIONS]

选项:
  --lines NUM      行数
  --follow, -f    实时跟踪
```

## 配置文件

### 配置文件位置

- Linux/MacOS: `~/.dify/config.yml`
- Windows: `%USERPROFILE%\.dify\config.yml`

### 配置文件格式

```yaml
api_key: your-api-key
base_url: https://api.dify.ai/v1
timeout: 30
retry:
  max_attempts: 3
  backoff: 2
log_level: info
```

## 完整示例

```bash
# 1. 配置 CLI
dify config set api-key app-xxx

# 2. 创建插件
dify plugin init my-plugin --template tool
cd my-plugin

# 3. 开发插件
dify plugin run --debug

# 4. 打包插件
dify plugin package -o ../dist

# 5. 部署工作流
dify workflow run wf_xxx -i '{"query": "hello"}' --watch

# 6. 查看日志
dify logs --follow
```

## 退出码

| 退出码 | 说明 |
|--------|------|
| 0 | 成功 |
| 1 | 一般错误 |
| 2 | 参数错误 |
| 3 | 认证错误 |
| 4 | 权限错误 |
| 5 | 找不到资源 |

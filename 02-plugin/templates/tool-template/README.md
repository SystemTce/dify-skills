# Tool 插件模板

这是一个简单的 Tool 插件模板，可以作为开发新插件的起点。

## 使用方法

1. 复制此模板到新目录
2. 修改 `manifest.yaml` 中的插件信息
3. 实现 `tools/my_tool.py` 中的逻辑
4. 测试和打包

## 文件结构

```
tool-template/
├── manifest.yaml           # 插件清单
├── main.py                 # 入口文件
├── requirements.txt        # 依赖
├── provider/
│   ├── my_provider.yaml   # 提供者配置
│   └── my_provider.py     # 提供者实现
└── tools/
    ├── my_tool.yaml       # 工具配置
    └── my_tool.py         # 工具实现
```

## 快速开始

```bash
# 复制模板
cp -r tool-template my_new_tool

# 进入目录
cd my_new_tool

# 安装依赖
uv pip install -r requirements.txt

# 运行插件
dify plugin run
```

## 自定义

### 1. 修改插件信息

编辑 `manifest.yaml`：
- `name`: 插件名称
- `author`: 作者名称
- `description`: 插件描述

### 2. 实现工具逻辑

编辑 `tools/my_tool.py`：
- 修改 `_invoke` 方法
- 添加业务逻辑

### 3. 配置参数

编辑 `tools/my_tool.yaml`：
- 添加工具参数
- 配置参数类型和验证

## 示例

参考 Google Translate 插件的实现。

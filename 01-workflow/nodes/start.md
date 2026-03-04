# Start 节点详解

## 概述

Start 节点是工作流的入口点，定义工作流的输入变量和初始配置。每个工作流必须有且只有一个 Start 节点。

## 核心功能

- 定义工作流输入变量
- 配置文件上传功能
- 设置初始参数
- 触发工作流执行

## 配置结构

### 基础配置

```yaml
- data:
    desc: '工作流入口'
    selected: false
    title: 开始
    type: start
    variables:
    - label: 用户输入
      max_length: 1000
      required: true
      type: text-input
      variable: user_input
  id: start
  position:
    x: 100
    y: 200
  type: custom
```

## 变量类型

### 1. 文本输入 (text-input)

**用途**: 接收用户的文本输入

**配置示例**:
```yaml
variables:
- label: 用户问题
  max_length: 2000
  required: true
  type: text-input
  variable: query
  description: 请输入您的问题
```

**参数说明**:
- `label`: 显示标签
- `max_length`: 最大字符数
- `required`: 是否必填
- `variable`: 变量名（用于引用）
- `description`: 提示信息

### 2. 段落文本 (paragraph)

**用途**: 接收多行文本输入

**配置示例**:
```yaml
variables:
- label: 文章内容
  max_length: 10000
  required: true
  type: paragraph
  variable: article
```

### 3. 下拉选择 (select)

**用途**: 从预定义选项中选择

**配置示例**:
```yaml
variables:
- label: 语言选择
  required: true
  type: select
  variable: language
  options:
  - 中文
  - English
  - 日本語
  default: 中文
```

### 4. 数字输入 (number)

**用途**: 接收数字输入

**配置示例**:
```yaml
variables:
- label: 数量
  required: true
  type: number
  variable: count
  default: 10
  min: 1
  max: 100
```

### 5. 文件上传 (file)

**用途**: 上传文件（图片、文档等）

**配置示例**:
```yaml
variables:
- label: 上传图片
  required: false
  type: file
  variable: image
  allowed_file_types:
  - image
  allowed_file_extensions:
  - .jpg
  - .png
  - .gif
```

## 系统变量

Start 节点自动提供以下系统变量：

```yaml
{{#sys.query#}}              # 用户查询（主输入）
{{#sys.files#}}              # 上传的文件列表
{{#sys.conversation_id#}}    # 会话ID
{{#sys.user_id#}}            # 用户ID
{{#sys.app_id#}}             # 应用ID
```

## 实际案例

### 案例1：简单问答

```yaml
- data:
    type: start
    variables:
    - label: 您的问题
      max_length: 500
      required: true
      type: text-input
      variable: question
  id: start
```

### 案例2：文档分析

```yaml
- data:
    type: start
    variables:
    - label: 上传文档
      required: true
      type: file
      variable: document
      allowed_file_types:
      - document
      allowed_file_extensions:
      - .pdf
      - .docx
      - .txt
    - label: 分析类型
      required: true
      type: select
      variable: analysis_type
      options:
      - 摘要
      - 关键词提取
      - 情感分析
  id: start
```

### 案例3：批量处理

```yaml
- data:
    type: start
    variables:
    - label: 数据列表
      required: true
      type: paragraph
      variable: data_list
      description: 每行一条数据
    - label: 并行数量
      required: true
      type: number
      variable: parallel_count
      default: 3
      min: 1
      max: 10
  id: start
```

## 最佳实践

### 1. 变量命名

- 使用有意义的变量名（如 `user_query` 而非 `input1`）
- 使用下划线分隔单词（snake_case）
- 避免使用保留关键字

### 2. 输入验证

- 设置合理的 `max_length` 限制
- 使用 `required` 标记必填字段
- 为数字输入设置 `min` 和 `max` 范围

### 3. 用户体验

- 提供清晰的 `label` 和 `description`
- 使用 `default` 值简化输入
- 合理使用 `select` 类型限制选项

### 4. 文件上传

- 明确指定 `allowed_file_types` 和 `allowed_file_extensions`
- 考虑文件大小限制
- 提供文件格式说明

## 常见问题

### Q: 如何引用 Start 节点的变量？

A: 使用语法 `{{#start.变量名#}}`，例如：
```yaml
{{#start.user_input#}}
{{#start.question#}}
```

### Q: 可以定义多少个输入变量？

A: 理论上没有限制，但建议不超过 10 个，以保持界面简洁。

### Q: 如何处理可选的输入？

A: 设置 `required: false`，并在后续节点中使用条件判断处理空值。

### Q: 文件上传后如何使用？

A: 文件会自动存储，可以通过 `{{#sys.files#}}` 引用文件列表，或在支持文件的节点（如 LLM 视觉模型）中直接使用。

## 相关节点

- [Answer 节点](./answer.md) - 输出工作流结果
- [LLM 节点](./llm.md) - 处理用户输入
- [Code 节点](./code.md) - 验证和转换输入

## 返回

[工作流 SKILL](../SKILL.md)

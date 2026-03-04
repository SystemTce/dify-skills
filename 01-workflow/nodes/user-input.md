# 用户输入节点详解

## 概述

用户输入节点用于收集终端用户的数据，是启动工作流和对话应用的入口点。它通过直接交互或 API 调用来按需执行应用。

## 核心功能

- 收集用户的文本、结构化数据和文件输入
- 支持多种输入类型（文本、下拉、数字、复选框、文件等）
- 提供预设变量用于工作流引用
- 支持 JSON 对象的 Schema 验证

## 输入类型

### 文本输入

**短文本**: 最多 256 个字符，适合简短输入
**段落文本**: 支持更长的多行文本输入

```yaml
variables:
- label: 用户问题
  type: text-input
  max_length: 256
  required: true
  variable: query
```

### 结构化输入

**下拉选择**:
```yaml
- label: 语言选择
  type: select
  variable: language
  options:
  - 中文
  - English
  - 日本語
```

**数字输入**:
```yaml
- label: 数量
  type: number
  variable: count
  min: 1
  max: 100
```

**复选框**:
```yaml
- label: 同意条款
  type: checkbox
  variable: agreed
```

**JSON 对象** (支持 Schema 验证):
```yaml
- label: 配置数据
  type: json
  variable: config
  schema:
    type: object
    properties:
      name:
        type: string
      age:
        type: number
    required: ["name"]
```

### 文件输入

**单文件上传**:
```yaml
- label: 上传文档
  type: file
  variable: document
  allowed_file_types:
  - document
```

**多文件上传**:
```yaml
- label: 上传图片
  type: files
  variable: images
  allowed_file_types:
  - image
  max_files: 5
```

## 预设变量

### userinput.files
用户上传的文件（工作流中的遗留变量，建议使用自定义文件字段）

### userinput.query
最新对话轮次的文本消息（仅对话流）

## 重要约束

- 每个应用画布只能包含一个用户输入节点
- 在对话流中，可以隐藏自定义字段但保持可引用性
- 必填字段不能被隐藏

## 文件处理

用户输入节点收集文件但不处理它们。下游节点必须适当处理内容：
- 文档 → 文档提取器节点
- 图片 → 支持视觉的 LLM 节点
- 结构化数据 → 代码节点进行解析

## 常见工作流

```yaml
# 典型流程
用户输入 → LLM 处理
用户输入 → 知识检索 → LLM 处理
用户输入 → If/Else 条件分支
```

## 实际案例

### 案例1：智能客服

```yaml
- data:
    type: user-input
    variables:
    - label: 您的问题
      type: text-input
      variable: question
      required: true
    - label: 问题类型
      type: select
      variable: category
      options:
      - 技术支持
      - 账户问题
      - 产品咨询
  id: user_input
```

### 案例2：文档分析

```yaml
- data:
    type: user-input
    variables:
    - label: 上传文档
      type: file
      variable: doc
      allowed_file_types:
      - document
      required: true
    - label: 分析深度
      type: select
      variable: depth
      options:
      - 快速摘要
      - 详细分析
      - 完整报告
  id: user_input
```

### 案例3：表单收集

```yaml
- data:
    type: user-input
    variables:
    - label: 姓名
      type: text-input
      variable: name
      required: true
    - label: 年龄
      type: number
      variable: age
      min: 18
      max: 100
    - label: 兴趣爱好
      type: paragraph
      variable: interests
    - label: 接收通知
      type: checkbox
      variable: subscribe
  id: user_input
```

## 最佳实践

### 1. 字段设计
- 只收集必要的信息
- 使用清晰的标签和描述
- 合理设置必填和可选字段

### 2. 类型选择
- 短输入用 text-input
- 长文本用 paragraph
- 限定选项用 select
- 数值用 number 并设置范围

### 3. 文件上传
- 明确指定允许的文件类型
- 设置合理的文件数量限制
- 提供文件格式说明

### 4. 验证策略
- 使用 JSON Schema 验证复杂数据
- 设置合理的长度和范围限制
- 在下游节点中进行二次验证

## 常见问题

### Q: 用户输入节点和 Start 节点有什么区别？

A: 用户输入节点是专门用于收集用户数据的节点，而 Start 节点是工作流的入口点。在某些情况下，它们可以是同一个节点。

### Q: 如何在对话流中隐藏某些字段？

A: 在对话流中，可以将自定义字段设置为隐藏，但它们仍然可以在工作流中引用。注意必填字段不能被隐藏。

### Q: 如何处理文件上传？

A: 用户输入节点只负责收集文件，实际处理需要在下游节点中进行：
- 使用文档提取器处理文档
- 使用支持视觉的 LLM 处理图片
- 使用代码节点处理结构化数据

## 相关节点

- [Start 节点](./start.md) - 工作流入口
- [文档提取器节点](./doc-extractor.md) - 处理上传的文档
- [LLM 节点](./llm.md) - 处理用户输入

## 返回

[工作流 SKILL](../SKILL.md)

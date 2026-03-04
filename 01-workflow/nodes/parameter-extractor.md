# 参数提取器节点详解

## 概述

参数提取器节点利用 LLM 将非结构化文本转换为结构化数据，在自然语言输入和结构化参数之间架起桥梁。

## 核心功能

- 智能文本解析
- 结构化数据提取
- 类型验证
- 错误处理

## 核心价值

将自然语言转换为工具、API 和其他工作流组件所需的结构化参数。

## 配置步骤

### 1. 选择输入变量
包含要提取文本的变量

### 2. 选择 LLM
选择具有强结构化输出能力的模型

### 3. 定义参数
指定名称、数据类型、描述和必需状态

### 4. 编写提取指令
为复杂参数提供清晰的说明和示例

## 提取方法

### Function Calling
- 可靠的提取
- 强类型合规性
- 推荐用于支持的模型

### Prompt-based
- 用于不支持函数调用的模型
- 基于提示的方法
- 需要更详细的指令

## 参数定义

```yaml
- data:
    type: parameter-extractor
    input: "{{#start.text#}}"
    model:
      provider: openai
      name: gpt-4
    parameters:
    - name: company_name
      type: string
      description: "公司名称"
      required: true
    - name: amount
      type: number
      description: "金额（数字）"
      required: true
    - name: date
      type: string
      description: "日期（YYYY-MM-DD 格式）"
      required: false
    instructions: |
      从文本中提取公司名称、金额和日期。
      如果日期不明确，可以留空。
  id: extract_params
```

## 支持的数据类型

- **string**: 文本字符串
- **number**: 数值
- **boolean**: 布尔值
- **object**: JSON 对象
- **array**: 数组

## 输出变量

```yaml
{{#extract_params.参数名#}}        # 提取的参数值
{{#extract_params.__is_success#}}  # 提取是否成功
{{#extract_params.__reason#}}      # 失败原因
```

## 实际案例

### 案例1：工具参数准备

```yaml
# 提取搜索参数
- data:
    type: parameter-extractor
    input: "{{#sys.query#}}"
    model:
      provider: openai
      name: gpt-4
    parameters:
    - name: search_query
      type: string
      description: "搜索关键词"
      required: true
    - name: max_results
      type: number
      description: "最大结果数（1-10）"
      required: false
    instructions: |
      从用户查询中提取搜索关键词。
      如果用户指定了结果数量，提取该数字。
  id: extract_search_params

# 调用搜索工具
- data:
    type: tool
    tool_name: google_search
    parameters:
      query: "{{#extract_search_params.search_query#}}"
      num_results: "{{#extract_search_params.max_results#}}"
  id: search
```

### 案例2：表单数据提取

```yaml
# 从自由文本提取表单数据
- data:
    type: parameter-extractor
    input: "{{#start.user_message#}}"
    model:
      provider: anthropic
      name: claude-3-opus
    parameters:
    - name: name
      type: string
      description: "用户姓名"
      required: true
    - name: email
      type: string
      description: "电子邮件地址"
      required: true
    - name: phone
      type: string
      description: "电话号码"
      required: false
    - name: message
      type: string
      description: "留言内容"
      required: true
    instructions: |
      从用户消息中提取联系信息和留言。
      邮箱格式：xxx@xxx.com
      电话格式：可选，任何格式
  id: extract_form

# 检查提取是否成功
- data:
    type: if-else
    conditions:
    - variable: "{{#extract_form.__is_success#}}"
      operator: "equals"
      value: true
  id: check_extraction

# 保存数据
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/contacts"
    body:
      name: "{{#extract_form.name#}}"
      email: "{{#extract_form.email#}}"
      phone: "{{#extract_form.phone#}}"
      message: "{{#extract_form.message#}}"
  id: save_contact
```

### 案例3：API 数据构建

```yaml
# 提取 API 参数
- data:
    type: parameter-extractor
    input: "{{#sys.query#}}"
    model:
      provider: openai
      name: gpt-4
    parameters:
    - name: start_date
      type: string
      description: "开始日期（YYYY-MM-DD）"
      required: true
    - name: end_date
      type: string
      description: "结束日期（YYYY-MM-DD）"
      required: true
    - name: category
      type: string
      description: "类别（sales, marketing, support）"
      required: true
    - name: format
      type: string
      description: "输出格式（json, csv, pdf）"
      required: false
    instructions: |
      从查询中提取报告参数。
      日期必须是 YYYY-MM-DD 格式。
      类别必须是：sales, marketing, 或 support。
      如果未指定格式，默认为 json。
  id: extract_report_params

# 调用 API
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/reports"
    body:
      start_date: "{{#extract_report_params.start_date#}}"
      end_date: "{{#extract_report_params.end_date#}}"
      category: "{{#extract_report_params.category#}}"
      format: "{{#extract_report_params.format#}}"
  id: generate_report
```

## 使用场景

### 工具参数准备
- 搜索查询提取
- API 参数构建
- 函数调用准备
- 命令参数解析

### 格式转换
- 文本到结构化数据
- 自然语言到 JSON
- 非结构化到结构化
- 格式标准化

### 表单处理
- 自由文本表单提取
- 对话式表单填写
- 数据验证
- 字段映射

### 数据提取
- 文档信息提取
- 实体识别
- 关键信息抽取
- 元数据生成

## 最佳实践

### 1. 参数描述
- 提供清晰的描述
- 说明格式要求
- 给出示例值
- 指定约束条件

### 2. 提取指令
- 编写详细的指令
- 包含示例
- 说明边缘情况
- 提供格式规范

### 3. 数据类型
- 选择合适的类型
- 确保兼容性
- 验证类型转换
- 处理类型错误

### 4. 错误处理
- 检查 `__is_success`
- 处理提取失败
- 提供默认值
- 记录失败原因

### 5. 模型选择
- 使用支持函数调用的模型
- 选择理解能力强的模型
- 考虑成本和速度
- 测试提取准确性

## 常见问题

### Q: 如何提高提取准确性？

A:
1. 提供清晰的参数描述
2. 在指令中包含示例
3. 使用更强大的模型
4. 指定格式要求
5. 添加验证步骤

### Q: 提取失败怎么办？

A:
1. 检查 `__is_success` 变量
2. 查看 `__reason` 了解原因
3. 实现重试逻辑
4. 提供降级方案
5. 要求用户重新输入

### Q: 如何处理可选参数？

A:
- 设置 `required: false`
- 提供默认值
- 在下游节点检查是否存在
- 使用 If-Else 处理

### Q: 支持复杂对象吗？

A:
- 支持嵌套对象
- 支持数组
- 需要详细的 schema 描述
- 可能需要多步提取

## 相关节点

- [LLM 节点](./llm.md) - 语言模型
- [Tool 节点](./tools.md) - 工具调用
- [HTTP Request 节点](./http-request.md) - API 调用
- [Variable Assigner 节点](./variable-assigner.md) - 变量赋值

## 返回

[工作流 SKILL](../SKILL.md)

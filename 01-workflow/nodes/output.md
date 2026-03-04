# 输出节点详解

## 概述

输出 (Output) 节点定义工作流端点并指定返回给终端用户的数据。它以前称为结束节点，现在在工作流中是可选的。

## 核心功能

- 定义工作流返回值
- 指定输出变量
- 支持多种数据类型
- API 接口返回

## 重要说明

- 输出节点仅适用于工作流应用 (Workflow)
- 对于对话流 (Chatflow)，使用答案节点处理响应
- 作为后端 API 暴露时，没有输出节点的工作流不会返回任何值

## 配置结构

### 基础配置

```yaml
- data:
    type: output
    outputs:
    - variable_selector: ["llm", "text"]
      output_name: result
      output_type: string
  id: output
```

### 多输出配置

```yaml
- data:
    type: output
    outputs:
    - variable_selector: ["llm", "text"]
      output_name: answer
      output_type: string
    - variable_selector: ["code", "data"]
      output_name: processed_data
      output_type: object
    - variable_selector: ["http", "status"]
      output_name: status_code
      output_type: number
  id: output
```

## 输出变量配置

### 字符串输出

```yaml
outputs:
- variable_selector: ["llm", "text"]
  output_name: response
  output_type: string
```

### 数字输出

```yaml
outputs:
- variable_selector: ["code", "count"]
  output_name: total_count
  output_type: number
```

### 对象输出

```yaml
outputs:
- variable_selector: ["code", "result"]
  output_name: data
  output_type: object
```

### 数组输出

```yaml
outputs:
- variable_selector: ["iteration", "results"]
  output_name: items
  output_type: array
```

### 文件输出

```yaml
outputs:
- variable_selector: ["doc_extractor", "file"]
  output_name: document
  output_type: file
```

## 实际案例

### 案例1：简单文本输出

```yaml
- data:
    type: output
    outputs:
    - variable_selector: ["llm", "text"]
      output_name: result
      output_type: string
  id: output
```

**API 返回**:
```json
{
  "result": "这是 LLM 生成的文本"
}
```

### 案例2：多字段输出

```yaml
- data:
    type: output
    outputs:
    - variable_selector: ["llm", "text"]
      output_name: answer
      output_type: string
    - variable_selector: ["knowledge", "result"]
      output_name: references
      output_type: array
    - variable_selector: ["code", "confidence"]
      output_name: confidence_score
      output_type: number
  id: output
```

**API 返回**:
```json
{
  "answer": "基于知识库的回答",
  "references": [
    {"title": "文档1", "content": "..."},
    {"title": "文档2", "content": "..."}
  ],
  "confidence_score": 0.95
}
```

### 案例3：结构化数据输出

```yaml
- data:
    type: output
    outputs:
    - variable_selector: ["code", "analysis_result"]
      output_name: analysis
      output_type: object
  id: output
```

**API 返回**:
```json
{
  "analysis": {
    "summary": "数据分析摘要",
    "metrics": {
      "total": 100,
      "success": 95,
      "failed": 5
    },
    "recommendations": [
      "建议1",
      "建议2"
    ]
  }
}
```

### 案例4：文件输出

```yaml
- data:
    type: output
    outputs:
    - variable_selector: ["report_generator", "file"]
      output_name: report
      output_type: file
    - variable_selector: ["llm", "summary"]
      output_name: summary
      output_type: string
  id: output
```

**API 返回**:
```json
{
  "report": {
    "url": "https://...",
    "filename": "report.pdf",
    "size": 1024000
  },
  "summary": "报告摘要"
}
```

### 案例5：批处理输出

```yaml
- data:
    type: output
    outputs:
    - variable_selector: ["iteration", "results"]
      output_name: processed_items
      output_type: array
    - variable_selector: ["code", "total_count"]
      output_name: total
      output_type: number
    - variable_selector: ["code", "success_count"]
      output_name: success
      output_type: number
    - variable_selector: ["code", "failed_count"]
      output_name: failed
      output_type: number
  id: output
```

**API 返回**:
```json
{
  "processed_items": [
    {"id": 1, "status": "success", "result": "..."},
    {"id": 2, "status": "success", "result": "..."},
    {"id": 3, "status": "failed", "error": "..."}
  ],
  "total": 3,
  "success": 2,
  "failed": 1
}
```

## 最佳实践

### 1. 输出设计
- 使用有意义的输出名称
- 选择正确的数据类型
- 提供完整的必要信息
- 保持输出结构一致

### 2. 数据类型
- 文本内容 → string
- 数值数据 → number
- 复杂结构 → object
- 列表数据 → array
- 文件资源 → file

### 3. API 设计
- 遵循 RESTful 规范
- 提供清晰的字段命名
- 包含状态和错误信息
- 考虑版本兼容性

### 4. 性能优化
- 避免输出过大的数据
- 使用分页处理大量数据
- 压缩文件输出
- 缓存常用结果

### 5. 错误处理
- 包含错误状态字段
- 提供详细的错误信息
- 使用标准的错误码
- 记录失败原因

## 输出节点 vs 答案节点

| 特性 | 输出节点 | 答案节点 |
|------|---------|---------|
| 应用类型 | 工作流 (Workflow) | 对话流 (Chatflow) |
| 用途 | API 返回值 | 用户界面显示 |
| 数据格式 | 结构化 JSON | 文本/多模态 |
| 是否必需 | 可选 | 可选 |
| 多个节点 | 通常一个 | 可以多个 |

## 常见问题

### Q: 输出节点是必需的吗？

A: 不是必需的，但如果工作流作为 API 暴露，没有输出节点将不会返回任何值。

### Q: 可以有多个输出节点吗？

A: 技术上可以，但通常一个工作流只有一个输出节点。如果有多个分支，使用变量聚合器合并结果后再输出。

### Q: 如何输出复杂的数据结构？

A: 使用代码节点构建复杂的对象或数组，然后在输出节点中引用：
```yaml
outputs:
- variable_selector: ["code", "complex_data"]
  output_name: data
  output_type: object
```

### Q: 输出节点和 Answer 节点有什么区别？

A:
- 输出节点：用于工作流，返回结构化数据给 API 调用者
- 答案节点：用于对话流，向用户显示格式化的内容

### Q: 如何处理输出中的敏感信息？

A:
1. 在代码节点中过滤敏感字段
2. 使用条件分支控制输出内容
3. 实现数据脱敏逻辑
4. 在 API 层面添加权限控制

## 相关节点

- [答案节点](./answer.md) - 对话流输出
- [Code 节点](./code.md) - 构建输出数据
- [变量聚合节点](./variable-aggregator.md) - 合并多分支输出

## 返回

[工作流 SKILL](../SKILL.md)

# 答案节点详解

## 概述

答案 (Answer) 节点是对话流应用专用的组件，用于向用户传递内容。它可以格式化响应、组合文本与变量，并流式传输包括文本、图片和文件在内的多模态内容。

## 核心功能

- 格式化响应内容
- 组合静态文本和动态变量
- 支持多模态内容（文本、图片、文件）
- 流式输出
- Markdown 格式支持

## 配置结构

### 基础配置

```yaml
- data:
    type: answer
    answer: "{{#llm.text#}}"
  id: answer
```

### 多模态配置

```yaml
- data:
    type: answer
    answer: |
      ## 分析结果

      {{#llm.text#}}

      相关图片：
      {{#tool.images#}}

      附件文档：
      {{#doc_processor.files#}}
  id: multimodal_answer
```

## 内容配置

### 静态文本

```yaml
answer: "感谢您的提问！我们会尽快回复。"
```

### 变量引用

使用 `{{variable_name}}` 语法引用节点输出：

```yaml
answer: |
  用户问题：{{#start.query#}}

  AI 回答：{{#llm.text#}}

  参考来源：{{#knowledge.result#}}
```

### Markdown 格式

```yaml
answer: |
  # 分析报告

  ## 摘要
  {{#llm.summary#}}

  ## 详细内容
  {{#llm.details#}}

  ## 建议
  - {{#llm.suggestion1#}}
  - {{#llm.suggestion2#}}
```

## 多模态支持

### 文本内容

支持 Markdown 格式：

```yaml
answer: |
  **重要提示**：{{#llm.notice#}}

  *分析结果*：{{#llm.result#}}

  `代码示例`：{{#code.output#}}
```

### 图片内容

从工具、上传或工作流处理中获取：

```yaml
answer: |
  生成的图片：

  {{#image_tool.images#}}

  说明：{{#llm.description#}}
```

### 文件内容

文档、电子表格等：

```yaml
answer: |
  处理完成！

  下载文件：{{#file_processor.files#}}

  文件说明：{{#llm.file_description#}}
```

## 多节点使用

### 渐进式响应

立即确认，同时继续处理：

```yaml
# 第一个答案节点 - 立即响应
- data:
    type: answer
    answer: "正在处理您的请求，请稍候..."
  id: answer_1

# 处理节点
- data:
    type: llm
    # ... LLM 配置
  id: llm

# 第二个答案节点 - 最终结果
- data:
    type: answer
    answer: "处理完成：{{#llm.text#}}"
  id: answer_2
```

### 条件响应

根据分支逻辑返回不同内容：

```yaml
# If 分支 - 成功
- data:
    type: answer
    answer: "✓ 操作成功：{{#process.result#}}"
  id: success_answer

# Else 分支 - 失败
- data:
    type: answer
    answer: "✗ 操作失败：{{#process.error#}}"
  id: failure_answer
```

### 流式更新

随着结果可用逐步传递：

```yaml
# 答案 1 - 初步结果
- data:
    type: answer
    answer: "初步分析：{{#llm1.text#}}"
  id: answer_1

# 答案 2 - 详细结果
- data:
    type: answer
    answer: "详细分析：{{#llm2.text#}}"
  id: answer_2

# 答案 3 - 最终建议
- data:
    type: answer
    answer: "建议方案：{{#llm3.text#}}"
  id: answer_3
```

## 变量集成

### LLM 输出

```yaml
answer: "{{#llm.text#}}"
```

### 知识检索结果

```yaml
answer: |
  基于知识库的回答：

  {{#knowledge.result#}}
```

### 外部工具/API

```yaml
answer: |
  查询结果：

  {{#http_request.body#}}
```

### 文件处理节点

```yaml
answer: |
  文档内容：

  {{#doc_extractor.text#}}

  附件：{{#doc_extractor.files#}}
```

## 实际案例

### 案例1：简单问答

```yaml
- data:
    type: answer
    answer: "{{#llm.text#}}"
  id: answer
```

### 案例2：格式化响应

```yaml
- data:
    type: answer
    answer: |
      # 智能助手回复

      ## 您的问题
      {{#start.query#}}

      ## 我的回答
      {{#llm.text#}}

      ## 参考资料
      {{#knowledge.result#}}

      ---
      *如有其他问题，请随时提问*
  id: formatted_answer
```

### 案例3：多模态响应

```yaml
- data:
    type: answer
    answer: |
      ## 图片分析结果

      {{#vision_llm.text#}}

      ### 原始图片
      {{#start.image#}}

      ### 标注图片
      {{#image_tool.annotated_image#}}

      ### 详细报告
      下载完整报告：{{#report_generator.file#}}
  id: multimodal_answer
```

### 案例4：渐进式响应

```yaml
# 立即响应
- data:
    type: answer
    answer: "🔍 正在分析您的文档..."
  id: answer_processing

# 中间结果
- data:
    type: answer
    answer: |
      📊 初步分析完成

      文档类型：{{#classifier.type#}}
      页数：{{#doc_extractor.page_count#}}
  id: answer_preliminary

# 最终结果
- data:
    type: answer
    answer: |
      ✅ 分析完成

      ## 摘要
      {{#llm_summary.text#}}

      ## 关键信息
      {{#llm_extract.text#}}

      ## 完整报告
      {{#report.file#}}
  id: answer_final
```

### 案例5：条件响应

```yaml
# 成功分支
- data:
    type: answer
    answer: |
      ✓ 订单创建成功

      订单号：{{#api.order_id#}}
      金额：{{#api.amount#}}
      预计送达：{{#api.delivery_date#}}
  id: success_answer

# 失败分支
- data:
    type: answer
    answer: |
      ✗ 订单创建失败

      错误原因：{{#api.error_message#}}

      请检查以下信息：
      - 商品库存
      - 配送地址
      - 支付方式
  id: failure_answer
```

## 最佳实践

### 1. 内容设计
- 使用清晰的结构（标题、段落）
- 提供有用的上下文信息
- 使用 Markdown 增强可读性
- 包含必要的说明和提示

### 2. 变量使用
- 验证变量是否存在
- 提供默认值或错误处理
- 格式化变量输出
- 避免暴露敏感信息

### 3. 多模态内容
- 为图片提供描述
- 为文件提供下载说明
- 合理组织不同类型的内容
- 考虑移动端显示

### 4. 用户体验
- 提供即时反馈
- 使用渐进式响应
- 清晰的成功/失败提示
- 友好的错误消息

### 5. 性能考虑
- 避免过长的响应
- 使用流式输出
- 优化图片大小
- 压缩大文件

## 常见问题

### Q: 答案节点和输出节点有什么区别？

A:
- 答案节点：用于对话流 (Chatflow)，向用户显示内容
- 输出节点：用于工作流 (Workflow)，定义 API 返回值

### Q: 可以在一个工作流中使用多个答案节点吗？

A: 可以。多个答案节点可以实现：
- 渐进式响应（逐步显示结果）
- 条件响应（不同分支不同内容）
- 流式更新（随着处理进度更新）

### Q: 如何在答案中显示图片？

A: 引用包含图片的变量：
```yaml
answer: "{{#image_variable#}}"
```

### Q: 如何格式化答案内容？

A: 使用 Markdown 语法：
```yaml
answer: |
  # 标题
  **粗体** *斜体*
  - 列表项
  `代码`
```

### Q: 答案节点支持哪些文件类型？

A: 支持所有常见文件类型：
- 文档：PDF, DOCX, TXT
- 图片：JPG, PNG, GIF
- 表格：XLSX, CSV
- 其他：ZIP, JSON 等

## 相关节点

- [输出节点](./output.md) - 工作流输出
- [LLM 节点](./llm.md) - 生成答案内容
- [模板节点](./template.md) - 格式化内容

## 返回

[工作流 SKILL](../SKILL.md)

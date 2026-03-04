# 列表操作符节点详解

## 概述

列表操作符节点通过过滤、排序和元素选择处理数组，解决大多数节点期望单个值而非数组的核心工作流挑战。

## 核心功能

- 数组过滤
- 元素排序
- 特定元素选择
- 类型路由

## 核心价值

将数组转换为可用的单个值或过滤后的子集，使下游节点能够正确处理数据。

## 关键能力

### 过滤
按属性提取特定项目：
- 文件类型（image, document, audio, video）
- MIME 类型
- 扩展名
- 大小
- 名称
- 上传方法

### 排序
按任何属性升序（ASC）或降序（DESC）组织结果

### 选择
选择特定元素：
- 前 N 项（1-20）
- 仅第一条记录
- 仅最后一条记录

## 支持的数据类型

```yaml
- array[string]
- array[number]
- array[file]
- array[boolean]
```

## 配置方法

### 基础过滤

```yaml
- data:
    type: list-operator
    input: "{{#start.files#}}"
    filter:
      type: file_type
      value: image
  id: filter_images
```

### 排序和选择

```yaml
- data:
    type: list-operator
    input: "{{#start.items#}}"
    sort:
      field: created_at
      order: DESC
    limit: 5
  id: recent_items
```

### 组合操作

```yaml
- data:
    type: list-operator
    input: "{{#start.files#}}"
    filter:
      type: file_type
      value: document
    sort:
      field: size
      order: DESC
    select: first_record
  id: largest_document
```

## 输出变量

```yaml
{{#list_op.result#}}         # 完整的过滤和排序数组
{{#list_op.first_record#}}   # 第一个元素
{{#list_op.last_record#}}    # 最后一个元素
```

## 实际案例

### 案例1：混合文件处理

```yaml
# 用户上传多个文件
- data:
    type: user-input
    variables:
    - label: 上传文件
      type: files
      variable: files
  id: user_input

# 过滤图片
- data:
    type: list-operator
    input: "{{#user_input.files#}}"
    filter:
      type: file_type
      value: image
  id: filter_images

# 过滤文档
- data:
    type: list-operator
    input: "{{#user_input.files#}}"
    filter:
      type: file_type
      value: document
  id: filter_documents

# 处理图片（使用视觉 LLM）
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4-vision
    prompt_template:
    - role: user
      text: "分析这些图片"
      files: "{{#filter_images.result#}}"
  id: analyze_images

# 处理文档（使用文档提取器）
- data:
    type: doc-extractor
    input: "{{#filter_documents.result#}}"
  id: extract_documents
```

### 案例2：获取最新文件

```yaml
# 按时间排序并获取最新
- data:
    type: list-operator
    input: "{{#start.files#}}"
    sort:
      field: upload_time
      order: DESC
    select: first_record
  id: latest_file

# 处理最新文件
- data:
    type: doc-extractor
    input: "{{#latest_file.first_record#}}"
  id: process_latest
```

### 案例3：按大小过滤

```yaml
# 过滤大文件
- data:
    type: list-operator
    input: "{{#start.files#}}"
    filter:
      type: size
      operator: greater_than
      value: 1048576  # 1MB
    sort:
      field: size
      order: DESC
  id: large_files

# 获取最大的文件
- data:
    type: list-operator
    input: "{{#large_files.result#}}"
    select: first_record
  id: largest_file
```

### 案例4：按扩展名过滤

```yaml
# 过滤 PDF 文件
- data:
    type: list-operator
    input: "{{#start.files#}}"
    filter:
      type: extension
      value: pdf
  id: pdf_files

# 迭代处理所有 PDF
- data:
    type: iteration
    input_variable: "{{#pdf_files.result#}}"
  id: process_pdfs
```

## 使用场景

### 混合文件处理
- 分离图片和文档
- 路由到不同处理器
- 多模态内容处理
- 文件类型分类

### 数据筛选
- 按条件过滤
- 大小限制
- 类型选择
- 属性匹配

### 排序和选择
- 获取最新/最旧
- 最大/最小值
- Top N 选择
- 优先级排序

### 批量处理
- 预处理数组
- 数据清洗
- 子集提取
- 分组处理

## 最佳实践

### 1. 过滤策略
- 明确过滤条件
- 处理空结果
- 验证过滤结果
- 记录过滤逻辑

### 2. 排序选择
- 选择合适的排序字段
- 确定排序方向
- 考虑性能影响
- 处理相同值

### 3. 元素选择
- first_record: 最新/最重要
- last_record: 最旧/最后
- top_n: 批量处理
- 根据场景选择

### 4. 错误处理
- 检查空数组
- 验证数据类型
- 处理过滤失败
- 提供降级方案

## 常见问题

### Q: 如何处理空结果？

A:
1. 使用 If-Else 检查结果
2. 提供默认值
3. 显示友好提示
4. 实现降级逻辑

### Q: 可以组合多个过滤条件吗？

A:
- 使用多个列表操作符节点串联
- 每个节点一个过滤条件
- 按顺序应用过滤
- 最终得到交集结果

### Q: 如何获取中间元素？

A:
- 使用排序和 top_n
- 通过代码节点自定义选择
- 使用迭代节点处理
- 考虑使用数组切片

### Q: 支持自定义过滤逻辑吗？

A:
- 列表操作符提供预定义过滤
- 复杂逻辑使用代码节点
- 可以组合多个操作符
- 考虑性能和可维护性

## 相关节点

- [Iteration 节点](./iteration.md) - 批量处理
- [Code 节点](./code.md) - 自定义逻辑
- [Doc Extractor 节点](./doc-extractor.md) - 文档处理
- [LLM 节点](./llm.md) - 内容分析

## 返回

[工作流 SKILL](../SKILL.md)

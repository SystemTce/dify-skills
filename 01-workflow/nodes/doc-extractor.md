# 文档提取器节点详解

## 概述

文档提取器节点将上传的文件转换为大语言模型可以处理的文本，作为文件上传和 AI 分析之间的桥梁。

## 核心功能

- 多格式文件解析
- 文本内容提取
- 结构保留
- 批量处理支持

## 支持的文件类型

### 文本格式
- **TXT**: 纯文本文件
- **Markdown**: Markdown 格式文档
- **HTML**: 网页文件

### Office 文档
- **DOCX**: Word 文档（直接解析）
- **DOC**: 旧版 Word（需要 Unstructured API）
- **Excel**: 转换为 Markdown 表格
- **CSV**: 转换为 Markdown 表格
- **PowerPoint**: 需要 Unstructured API

### PDF 文件
- 基于文本的 PDF（使用 pypdfium2）
- 扫描版 PDF（需要 OCR）

### 专用格式
- **EPUB**: 电子书格式
- **VTT**: 字幕文件
- **JSON/YAML**: 结构化数据
- **Properties**: 配置文件
- **EML/MSG**: 邮件格式

### 不支持的格式
- 图片文件（需要外部工具）
- 音频文件（需要外部工具）
- 视频文件（需要外部工具）

## 配置方法

### 单文件提取

```yaml
- data:
    type: doc-extractor
    input:
      variable: "{{#start.document#}}"
  id: extract_doc
```

### 多文件提取

```yaml
- data:
    type: doc-extractor
    input:
      variable: "{{#start.documents#}}"
  id: extract_docs
```

## 输出变量

### 单文件输出

```yaml
{{#extract_doc.text#}}          # 提取的文本内容
{{#extract_doc.file_name#}}     # 文件名
{{#extract_doc.file_type#}}     # 文件类型
{{#extract_doc.file_size#}}     # 文件大小
```

### 多文件输出

```yaml
{{#extract_docs.text#}}         # 文本数组
{{#extract_docs.files#}}        # 文件信息数组
```

## 实际案例

### 案例1：ChatPDF 应用

```yaml
# 用户上传文档
- data:
    type: user-input
    variables:
    - label: 上传 PDF
      type: file
      variable: pdf_file
      allowed_file_types:
      - document
  id: user_input

# 提取文档内容
- data:
    type: doc-extractor
    input:
      variable: "{{#user_input.pdf_file#}}"
  id: extract_pdf

# 使用 LLM 分析
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: system
      text: "你是文档分析助手"
    - role: user
      text: |
        文档内容：
        {{#extract_pdf.text#}}

        用户问题：{{#sys.query#}}
  id: analyze_doc
```

### 案例2：合同信息提取

```yaml
# 提取合同文档
- data:
    type: doc-extractor
    input:
      variable: "{{#start.contract#}}"
  id: extract_contract

# 提取关键信息
- data:
    type: parameter-extractor
    input: "{{#extract_contract.text#}}"
    parameters:
    - name: party_a
      type: string
      description: "甲方名称"
    - name: party_b
      type: string
      description: "乙方名称"
    - name: amount
      type: number
      description: "合同金额"
    - name: start_date
      type: string
      description: "开始日期"
    - name: end_date
      type: string
      description: "结束日期"
  id: extract_info

# 生成摘要
- data:
    type: llm
    model:
      provider: anthropic
      name: claude-3-opus
    prompt_template:
    - role: user
      text: |
        根据以下信息生成合同摘要：
        
        甲方：{{#extract_info.party_a#}}
        乙方：{{#extract_info.party_b#}}
        金额：{{#extract_info.amount#}}
        期限：{{#extract_info.start_date#}} 至 {{#extract_info.end_date#}}
        
        原文：
        {{#extract_contract.text#}}
  id: generate_summary
```

### 案例3：批量文档处理

```yaml
# 提取多个文档
- data:
    type: doc-extractor
    input:
      variable: "{{#start.documents#}}"
  id: extract_all

# 迭代处理每个文档
- data:
    type: iteration
    input_variable: "{{#extract_all.text#}}"
  id: process_each

# 分析单个文档
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: user
      text: |
        分析以下文档：
        {{#item#}}
        
        提取关键要点
  id: analyze_doc

# 聚合结果
- data:
    type: template
    template: |
      ## 文档分析报告
      
      {% for result in process_each.results %}
      ### 文档 {{ loop.index }}
      {{ result }}
      {% endfor %}
  id: final_report
```

### 案例4：研究论文分析

```yaml
# 提取论文
- data:
    type: doc-extractor
    input:
      variable: "{{#start.paper#}}"
  id: extract_paper

# 分段分析
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: user
      text: |
        分析论文的以下部分：
        
        {{#extract_paper.text#}}
        
        提取：
        1. 研究问题
        2. 方法论
        3. 主要发现
        4. 结论
  id: analyze_sections

# 生成综述
- data:
    type: llm
    model:
      provider: anthropic
      name: claude-3-opus
    prompt_template:
    - role: user
      text: |
        基于以下分析生成论文综述：
        
        {{#analyze_sections.text#}}
  id: generate_review
```

## 使用场景

### 文档问答
- PDF 文档查询
- 合同分析
- 报告解读
- 手册查询

### 信息提取
- 简历解析
- 发票处理
- 表单数据提取
- 证件识别

### 内容分析
- 文献综述
- 市场研究
- 竞品分析
- 舆情监测

### 批量处理
- 文档归档
- 数据迁移
- 格式转换
- 内容索引

## 最佳实践

### 1. 文件类型处理
- 明确支持的文件类型
- 提供格式说明
- 处理不支持的格式
- 实现格式验证

### 2. 大文件处理
- 分块处理长文档
- 使用摘要技术
- 实现进度提示
- 考虑超时限制

### 3. 表格处理
- Excel/CSV 转为 Markdown
- 保留表格结构
- 处理合并单元格
- 验证数据完整性

### 4. 编码处理
- 自动检测文件编码
- 处理特殊字符
- 统一编码格式
- 避免乱码问题

### 5. 错误处理
- 验证文件完整性
- 处理损坏文件
- 提供错误提示
- 实现降级方案

## 处理细节

### 文本提取
- 使用专用解析库
- 保留文档结构
- 提取元数据
- 清理格式标记

### 表格转换
- 转换为 Markdown 表格
- 保持列对齐
- 处理空单元格
- 支持多级表头

### 编码检测
- 自动识别编码
- 支持多种字符集
- 处理 BOM 标记
- 统一输出格式

## Unstructured API 配置

某些格式需要配置 Unstructured API：

```bash
# 环境变量配置
UNSTRUCTURED_API_URL=https://api.unstructured.io
UNSTRUCTURED_API_KEY=your_api_key
```

**需要 API 的格式**:
- DOC（旧版 Word）
- PowerPoint
- EPUB

## 常见问题

### Q: 如何处理扫描版 PDF？

A:
- 扫描版 PDF 需要 OCR 技术
- 使用外部 OCR 工具预处理
- 或使用支持 OCR 的文档提取服务
- 考虑使用视觉 LLM 直接处理

### Q: 提取的文本格式混乱怎么办？

A:
1. 检查原文件格式
2. 使用文本清理节点
3. 通过 LLM 重新格式化
4. 考虑使用专用解析工具

### Q: 如何处理大文件？

A:
1. 分块提取和处理
2. 使用摘要技术
3. 只提取关键部分
4. 考虑文件大小限制

### Q: 表格数据如何处理？

A:
- 提取为 Markdown 表格
- 使用代码节点解析
- 转换为结构化数据
- 通过 LLM 理解表格内容

### Q: 如何处理多语言文档？

A:
1. 自动检测语言
2. 使用支持多语言的模型
3. 必要时进行翻译
4. 保留原始语言信息

## 性能考虑

### 文件大小
- 小文件（< 1MB）：快速处理
- 中等文件（1-10MB）：正常处理
- 大文件（> 10MB）：考虑分块

### 处理时间
- 文本文件：毫秒级
- Office 文档：秒级
- PDF 文件：秒到分钟级
- 需要 API 的格式：取决于 API 响应

### 优化建议
- 缓存提取结果
- 并行处理多文件
- 使用更快的解析库
- 限制文件大小

## 相关节点

- [用户输入节点](./user-input.md) - 文件上传
- [LLM 节点](./llm.md) - 文本分析
- [参数提取器节点](./parameter-extractor.md) - 信息提取
- [迭代节点](./iteration.md) - 批量处理

## 返回

[工作流 SKILL](../SKILL.md)

# 知识检索节点详解

## 概述

知识检索节点将已有知识库集成到 Chatflow 或 Workflow 应用中，在指定知识库中检索与查询相关的信息，并将检索结果作为上下文内容传递给下游节点。

## 核心功能

- 多知识库检索
- 语义相似度搜索
- Rerank 重排序
- 元数据过滤
- 多模态检索（文本+图片）

## 典型工作流

```
用户输入 → 知识检索 → LLM 处理 → 输出回答
```

## 配置结构

### 基础配置

```yaml
- data:
    type: knowledge-retrieval
    dataset_ids:
    - dataset-id-1
    - dataset-id-2
    query_variable_selector: ["start", "sys.query"]
    retrieval_mode: single
    single_retrieval_config:
      model:
        provider: openai
        name: text-embedding-3-small
      top_k: 3
      score_threshold: 0.7
  id: knowledge
```

## 查询配置

### 文本查询

```yaml
query_variable_selector: ["userinput", "query"]
# 或
query_variable_selector: ["start", "sys.query"]
```

### 图片查询

支持多模态知识库（限制 2MB）：

```yaml
query_variable_selector: ["userinput", "image"]
# 图片会被转换为向量进行检索
```

## 知识库选择

### 单个知识库

```yaml
dataset_ids:
- dataset-abc123
```

### 多个知识库

同时检索并合并结果：

```yaml
dataset_ids:
- dataset-abc123
- dataset-def456
- dataset-ghi789
```

## 检索设置

### Rerank 模型

按相关性重新排序结果：

```yaml
single_retrieval_config:
  reranking_model:
    provider: cohere
    model: rerank-english-v2.0
  reranking_enabled: true
```

### Top K

返回的最大结果数：

```yaml
single_retrieval_config:
  top_k: 5  # 返回前5个最相关的结果
```

**建议值**:
- 简单问答：3-5
- 复杂分析：5-10
- 全面检索：10-20

### Score 阈值

相似度最低分数过滤：

```yaml
single_retrieval_config:
  score_threshold: 0.7  # 0.0-1.0
```

**建议值**:
- 高精度：0.8-1.0
- 平衡：0.6-0.8
- 高召回：0.4-0.6

### 权重设置

语义相似度与关键词匹配的平衡：

```yaml
single_retrieval_config:
  retrieval_method: hybrid
  semantic_weight: 0.7  # 语义相似度权重
  keyword_weight: 0.3   # 关键词匹配权重
```

## 元数据过滤

利用文档元数据限定检索范围：

```yaml
single_retrieval_config:
  metadata_filter:
    enabled: true
    filters:
    - field: category
      operator: equals
      value: "技术文档"
    - field: date
      operator: greater_than
      value: "2024-01-01"
```

**支持的操作符**:
- `equals`: 等于
- `not_equals`: 不等于
- `contains`: 包含
- `not_contains`: 不包含
- `greater_than`: 大于
- `less_than`: 小于

## 输出变量

### result 变量

包含分段内容、元数据、标题的文档数组：

```yaml
{{#knowledge.result#}}
```

**结构**:
```json
[
  {
    "content": "文档内容...",
    "metadata": {
      "title": "文档标题",
      "source": "来源",
      "category": "分类"
    },
    "score": 0.85
  }
]
```

### files 字段

若检索结果包含图片：

```yaml
{{#knowledge.files#}}
```

## 与 LLM 集成

在 LLM 节点的上下文字段中引用检索结果：

```yaml
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: system
      text: "你是一个知识库助手"
    - role: user
      text: |
        上下文信息：
        {{#knowledge.result#}}

        用户问题：{{#sys.query#}}

        请基于上下文回答问题。
    context:
      enabled: true
      variable_selector: ["knowledge", "result"]
    vision:
      enabled: true  # 处理图片
  id: llm
```

## 实际案例

### 案例1：简单 RAG 问答

```yaml
# 知识检索
- data:
    type: knowledge-retrieval
    dataset_ids:
    - dataset-faq
    query_variable_selector: ["start", "sys.query"]
    retrieval_mode: single
    single_retrieval_config:
      model:
        provider: openai
        name: text-embedding-3-small
      top_k: 3
      score_threshold: 0.7
  id: knowledge

# LLM 处理
- data:
    type: llm
    model:
      provider: openai
      name: gpt-3.5-turbo
    prompt_template:
    - role: system
      text: "基于知识库回答用户问题"
    - role: user
      text: |
        知识库内容：
        {{#knowledge.result#}}

        问题：{{#sys.query#}}
    context:
      enabled: true
      variable_selector: ["knowledge", "result"]
  id: llm
```

### 案例2：多知识库检索

```yaml
- data:
    type: knowledge-retrieval
    dataset_ids:
    - dataset-product-docs
    - dataset-user-manual
    - dataset-faq
    query_variable_selector: ["start", "sys.query"]
    retrieval_mode: single
    single_retrieval_config:
      model:
        provider: openai
        name: text-embedding-3-small
      top_k: 5
      score_threshold: 0.6
      reranking_enabled: true
      reranking_model:
        provider: cohere
        model: rerank-multilingual-v2.0
  id: multi_knowledge
```

### 案例3：带元数据过滤的检索

```yaml
- data:
    type: knowledge-retrieval
    dataset_ids:
    - dataset-docs
    query_variable_selector: ["start", "sys.query"]
    retrieval_mode: single
    single_retrieval_config:
      model:
        provider: openai
        name: text-embedding-3-small
      top_k: 5
      score_threshold: 0.7
      metadata_filter:
        enabled: true
        filters:
        - field: category
          operator: equals
          value: "{{#start.category#}}"
        - field: version
          operator: equals
          value: "latest"
  id: filtered_knowledge
```

### 案例4：多模态检索

```yaml
- data:
    type: knowledge-retrieval
    dataset_ids:
    - dataset-multimodal
    query_variable_selector: ["start", "sys.query"]
    retrieval_mode: single
    single_retrieval_config:
      model:
        provider: openai
        name: text-embedding-3-small
      top_k: 3
      score_threshold: 0.7
  id: multimodal_knowledge

# LLM 处理（支持视觉）
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4-vision-preview
    prompt_template:
    - role: user
      text: |
        检索到的内容：
        {{#multimodal_knowledge.result#}}

        图片：
        {{#multimodal_knowledge.files#}}

        问题：{{#sys.query#}}
    vision:
      enabled: true
  id: vision_llm
```

## 最佳实践

### 1. 知识库设计
- 合理分割文档（300-500 字/段）
- 添加有意义的元数据
- 定期更新知识库内容
- 使用清晰的文档标题

### 2. 检索优化
- 根据场景调整 Top K
- 使用 Rerank 提升精度
- 设置合理的 Score 阈值
- 利用元数据过滤缩小范围

### 3. 模型选择
- 中文内容：text-embedding-3-small
- 英文内容：text-embedding-ada-002
- 多语言：multilingual-e5-large
- 高精度：text-embedding-3-large

### 4. 成本优化
- 使用较小的 Embedding 模型
- 限制 Top K 数量
- 启用缓存机制
- 合理设置 Score 阈值

### 5. 质量提升
- 启用 Rerank 模型
- 使用混合检索（语义+关键词）
- 添加元数据过滤
- 定期评估检索质量

## 性能优化

### 检索速度
- 使用向量索引
- 限制知识库数量
- 优化文档分割
- 启用缓存

### 检索精度
- 使用 Rerank 模型
- 调整权重配置
- 优化 Score 阈值
- 改进文档质量

### 成本控制
- 选择合适的 Embedding 模型
- 限制 Top K 数量
- 减少 Rerank 调用
- 使用缓存策略

## 常见问题

### Q: 如何提升检索准确性？

A:
1. 启用 Rerank 模型
2. 调整 Score 阈值
3. 使用混合检索
4. 添加元数据过滤
5. 优化文档分割

### Q: 检索结果太少怎么办？

A:
1. 降低 Score 阈值
2. 增加 Top K 数量
3. 检查知识库内容
4. 调整检索权重

### Q: 如何处理多语言检索？

A:
1. 使用多语言 Embedding 模型
2. 分别建立不同语言的知识库
3. 使用多语言 Rerank 模型
4. 在 LLM 中指定输出语言

### Q: 如何优化检索成本？

A:
1. 使用较小的 Embedding 模型
2. 限制 Top K 数量
3. 减少 Rerank 使用
4. 启用缓存机制
5. 合理设置 Score 阈值

## 相关节点

- [LLM 节点](./llm.md) - 处理检索结果
- [If-Else 节点](./if-else.md) - 根据检索结果分支
- [Code 节点](./code.md) - 处理检索数据

## 返回

[工作流 SKILL](../SKILL.md)

# RAG 模式 (Retrieval-Augmented Generation)

## 概述

RAG (检索增强生成) 模式通过知识库检索为 LLM 提供相关上下文，提升回答的准确性和可靠性。

## 模式结构

```
Start → Knowledge Retrieval → LLM (with context) → Answer
```

## 适用场景

- 基于文档的问答系统
- 企业知识库助手
- 技术文档查询
- 客服机器人
- 专业领域问答

## 优势

- 提供准确的事实性回答
- 减少 LLM 幻觉
- 支持实时知识更新
- 可追溯信息来源
- 降低对模型能力的依赖

## 劣势

- 依赖知识库质量
- 检索准确性影响结果
- 增加系统复杂度
- 需要维护知识库

## 基础示例

### 示例1：简单 RAG 问答

```yaml
workflow:
  nodes:
  # 1. Start
  - data:
      type: start
      variables:
      - label: 用户问题
        type: text-input
        variable: question
    id: start

  # 2. Knowledge Retrieval
  - data:
      type: knowledge-retrieval
      dataset_ids:
      - "dataset-uuid-1"
      query_variable_selector: ["start", "question"]
      retrieval_mode: single
      single_retrieval_config:
        model:
          provider: openai
          name: text-embedding-3-small
        top_k: 3
        score_threshold: 0.7
    id: knowledge

  # 3. LLM with Context
  - data:
      type: llm
      model:
        provider: openai
        name: gpt-4
      prompt_template:
      - role: system
        text: |
          你是一个专业的知识库助手。
          请基于提供的上下文回答用户问题。
          如果上下文中没有相关信息，请明确告知用户。
      - role: user
        text: |
          上下文信息：
          {{#knowledge.result#}}

          用户问题：{{#start.question#}}
      context:
        enabled: true
        variable_selector: ["knowledge", "result"]
    id: llm

  # 4. Answer
  - data:
      type: answer
      answer: "{{#llm.text#}}"
    id: answer

  edges:
  - source: start
    target: knowledge
  - source: knowledge
    target: llm
  - source: llm
    target: answer
```

## 高级示例

### 示例2：多知识库 RAG

```yaml
workflow:
  nodes:
  - data:
      type: start
      variables:
      - label: 问题
        type: text-input
        variable: query
      - label: 知识库类型
        type: select
        variable: kb_type
        options: ["技术文档", "产品手册", "FAQ"]
    id: start

  # 条件选择知识库
  - data:
      type: if-else
      conditions:
      - variable_selector: ["start", "kb_type"]
        comparison_operator: is
        value: "技术文档"
    id: kb_selector

  # 技术文档知识库
  - data:
      type: knowledge-retrieval
      dataset_ids: ["tech-docs-uuid"]
      query_variable_selector: ["start", "query"]
      retrieval_mode: single
      single_retrieval_config:
        top_k: 5
        score_threshold: 0.75
    id: tech_kb

  # 产品手册知识库
  - data:
      type: knowledge-retrieval
      dataset_ids: ["product-manual-uuid"]
      query_variable_selector: ["start", "query"]
      retrieval_mode: single
      single_retrieval_config:
        top_k: 5
        score_threshold: 0.75
    id: product_kb

  # 聚合结果
  - data:
      type: variable-aggregator
      variables:
      - variable_selector: ["tech_kb", "result"]
      - variable_selector: ["product_kb", "result"]
    id: aggregator

  # LLM 生成答案
  - data:
      type: llm
      model:
        provider: anthropic
        name: claude-3-opus
      prompt_template:
      - role: user
        text: |
          基于以下知识库内容回答问题：
          {{#aggregator.output#}}

          问题：{{#start.query#}}
    id: llm

  - data:
      type: answer
      answer: "{{#llm.text#}}"
    id: answer
```

### 示例3：RAG + Reranking

```yaml
workflow:
  nodes:
  - data:
      type: start
      variables:
      - label: 查询
        type: text-input
        variable: query
    id: start

  # 初步检索（top_k 较大）
  - data:
      type: knowledge-retrieval
      dataset_ids: ["kb-uuid"]
      query_variable_selector: ["start", "query"]
      retrieval_mode: single
      single_retrieval_config:
        model:
          provider: openai
          name: text-embedding-3-small
        top_k: 10  # 初步检索更多结果
        score_threshold: 0.6
        # Reranking 配置
        reranking_model:
          provider: cohere
          model: rerank-english-v2.0
        reranking_mode: reranking_model
    id: knowledge

  # 验证检索结果
  - data:
      type: code
      code: |
        def main(result: str) -> dict:
            # 检查是否有有效结果
            has_result = len(result.strip()) > 0

            return {
                "has_result": has_result,
                "result": result
            }
      variables:
      - variable: result
        value_selector: ["knowledge", "result"]
    id: validate

  # 条件判断
  - data:
      type: if-else
      conditions:
      - variable_selector: ["validate", "has_result"]
        comparison_operator: is
        value: true
    id: check_result

  # 有结果：基于上下文回答
  - data:
      type: llm
      model:
        provider: openai
        name: gpt-4
      prompt_template:
      - role: system
        text: "你是一个专业的知识库助手"
      - role: user
        text: |
          上下文：{{#validate.result#}}
          问题：{{#start.query#}}
    id: llm_with_context

  # 无结果：通用回答
  - data:
      type: llm
      model:
        provider: openai
        name: gpt-3.5-turbo
      prompt_template:
      - role: user
        text: |
          抱歉，知识库中没有找到相关信息。
          请基于你的知识回答：{{#start.query#}}

          注意：这是基于通用知识的回答，可能不够准确。
    id: llm_fallback

  - data:
      type: answer
      answer: "{{#llm_with_context.text#}}{{#llm_fallback.text#}}"
    id: answer
```

### 示例4：对话式 RAG

```yaml
workflow:
  nodes:
  - data:
      type: start
      variables:
      - label: 用户消息
        type: text-input
        variable: message
    id: start

  # 提取查询意图
  - data:
      type: llm
      model:
        provider: openai
        name: gpt-3.5-turbo
      prompt_template:
      - role: system
        text: |
          你是一个查询意图提取助手。
          请从用户消息中提取核心查询关键词，用于知识库检索。
          只输出关键词，不要添加其他内容。
      - role: user
        text: "{{#start.message#}}"
      memory:
        window:
          enabled: true
          size: 5
    id: extract_query

  # 知识库检索
  - data:
      type: knowledge-retrieval
      dataset_ids: ["kb-uuid"]
      query_variable_selector: ["extract_query", "text"]
      retrieval_mode: single
      single_retrieval_config:
        top_k: 3
        score_threshold: 0.7
    id: knowledge

  # 生成回答（带对话记忆）
  - data:
      type: llm
      model:
        provider: openai
        name: gpt-4
      prompt_template:
      - role: system
        text: |
          你是一个友好的知识库助手。
          请基于提供的上下文和对话历史回答用户问题。
      - role: user
        text: |
          相关知识：
          {{#knowledge.result#}}

          用户消息：{{#start.message#}}
      context:
        enabled: true
        variable_selector: ["knowledge", "result"]
      memory:
        role_prefix:
          user: "用户"
          assistant: "助手"
        window:
          enabled: true
          size: 10
    id: llm

  - data:
      type: answer
      answer: "{{#llm.text#}}"
    id: answer
```

## 知识库配置

### 检索模式

#### Single (单次检索)

```yaml
retrieval_mode: single
single_retrieval_config:
  model:
    provider: openai
    name: text-embedding-3-small
  top_k: 3
  score_threshold: 0.7
```

#### Multiple (多次检索)

```yaml
retrieval_mode: multiple
multiple_retrieval_config:
  top_k: 2
  score_threshold: 0.5
```

### Reranking (重排序)

提升检索精度：

```yaml
reranking_model:
  provider: cohere
  model: rerank-english-v2.0
reranking_mode: reranking_model
```

## 性能优化

### 1. 检索参数调优

```yaml
# 平衡精度和召回率
top_k: 3-5              # 返回结果数量
score_threshold: 0.7    # 相似度阈值
```

### 2. 嵌入模型选择

- **轻量级**: text-embedding-ada-002
- **高质量**: text-embedding-3-small/large
- **多语言**: multilingual-e5-large

### 3. 知识库优化

- 文档分块大小：500-1000 字符
- 重叠区域：50-100 字符
- 定期更新和清理

### 4. 缓存策略

对常见问题启用缓存。

## 最佳实践

### 1. 提示词设计

```yaml
prompt_template:
- role: system
  text: |
    你是一个专业的知识库助手。

    规则：
    1. 仅基于提供的上下文回答
    2. 如果上下文不足，明确告知用户
    3. 引用具体的上下文内容
    4. 保持回答简洁准确

- role: user
  text: |
    上下文：
    {{#knowledge.result#}}

    问题：{{#start.query#}}

    请基于上下文回答问题。
```

### 2. 结果验证

```python
def main(result: str, query: str) -> dict:
    # 验证检索结果质量
    has_content = len(result.strip()) > 50
    is_relevant = query.lower() in result.lower()

    return {
        "result": result,
        "quality": "good" if (has_content and is_relevant) else "poor"
    }
```

### 3. 降级策略

```yaml
# 检索失败时的降级处理
If-Else:
  条件: 检索结果为空
  True: 使用通用 LLM 回答
  False: 使用 RAG 回答
```

### 4. 来源引用

```yaml
prompt_template:
- role: user
  text: |
    上下文：{{#knowledge.result#}}
    问题：{{#start.query#}}

    请回答问题，并在回答末尾注明信息来源。
```

## 常见问题

### Q: 如何提升检索准确性？

A:
1. 优化文档分块策略
2. 使用 Reranking 模型
3. 调整 `top_k` 和 `score_threshold`
4. 改进查询关键词提取

### Q: 如何处理多语言知识库？

A: 使用多语言嵌入模型（如 multilingual-e5）。

### Q: 如何减少 LLM 幻觉？

A:
1. 在提示词中强调"仅基于上下文"
2. 验证检索结果质量
3. 使用结构化输出
4. 添加来源引用

### Q: 如何优化成本？

A:
1. 使用轻量级嵌入模型
2. 合理设置 `top_k`
3. 启用缓存
4. 对简单问题使用轻量 LLM

## 相关模式

- [顺序管道模式](./sequential.md) - 基础流程
- [条件分支模式](./conditional.md) - 添加降级策略
- [多源聚合模式](./aggregation.md) - 整合多个知识库

## 返回

[工作流 SKILL](../SKILL.md)

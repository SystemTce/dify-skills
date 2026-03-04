# LLM 节点详解

## 概述

LLM (Large Language Model) 节点是工作流中最核心的节点之一，用于调用大语言模型进行文本生成、推理、分析等任务。

## 核心功能

- 调用各种 LLM 模型（GPT-4、Claude、Qwen 等）
- 支持系统提示词和用户提示词
- 集成知识库检索结果
- 支持对话记忆
- 支持视觉输入（多模态模型）
- 支持结构化输出

## 配置结构

### 基础配置

```yaml
- data:
    type: llm
    title: LLM
    model:
      provider: openai
      name: gpt-4
      mode: chat
      completion_params:
        temperature: 0.7
        max_tokens: 2000
        top_p: 1.0
        frequency_penalty: 0.0
        presence_penalty: 0.0
    prompt_template:
    - id: system
      role: system
      text: "你是一个专业的AI助手"
    - id: user
      role: user
      text: "{{#sys.query#}}"
  id: llm
```

## 模型配置

### 支持的模型提供商

- **OpenAI**: GPT-4, GPT-3.5-turbo
- **Anthropic**: Claude 3 Opus, Claude 3 Sonnet
- **Google**: Gemini Pro, Gemini Pro Vision
- **阿里云**: Qwen-Max, Qwen-Plus, Qwen-Turbo
- **智谱AI**: GLM-4, GLM-3-Turbo
- **DeepSeek**: DeepSeek-Chat, DeepSeek-Coder
- **本地模型**: Ollama, LocalAI

### 模型参数

#### temperature (温度)

**范围**: 0.0 - 2.0
**默认**: 0.7

- `0.0`: 确定性输出，适合事实性任务
- `0.7`: 平衡创造性和准确性
- `1.5+`: 高创造性，适合创意写作

**示例**:
```yaml
completion_params:
  temperature: 0.3  # 用于代码生成
```

#### max_tokens (最大令牌数)

**范围**: 1 - 模型上限
**默认**: 2000

控制生成文本的最大长度。

**示例**:
```yaml
completion_params:
  max_tokens: 4000  # 长文本生成
```

#### top_p (核采样)

**范围**: 0.0 - 1.0
**默认**: 1.0

控制输出的多样性。

**示例**:
```yaml
completion_params:
  top_p: 0.9  # 略微降低随机性
```

#### frequency_penalty (频率惩罚)

**范围**: -2.0 - 2.0
**默认**: 0.0

减少重复内容。

#### presence_penalty (存在惩罚)

**范围**: -2.0 - 2.0
**默认**: 0.0

鼓励谈论新话题。

## 提示词模板

### 系统消息 (System Message)

定义 AI 的角色和行为准则。

**示例**:
```yaml
prompt_template:
- id: system
  role: system
  text: |
    你是一个专业的技术文档写作助手。

    你的职责：
    1. 提供清晰、准确的技术说明
    2. 使用专业术语，但保持易懂
    3. 提供代码示例和最佳实践

    你的风格：
    - 简洁明了
    - 结构化输出
    - 注重实用性
```

### 用户消息 (User Message)

包含用户的实际输入和上下文。

**示例**:
```yaml
- id: user
  role: user
  text: |
    用户问题：{{#sys.query#}}

    相关背景：{{#knowledge.result#}}

    请基于以上信息回答用户问题。
```

### 变量引用

在提示词中引用其他节点的输出：

```yaml
text: |
  用户输入：{{#start.user_input#}}
  检索结果：{{#knowledge.result#}}
  代码输出：{{#code.output#}}
  上一步结果：{{#llm1.text#}}
```

## 上下文配置

### 知识库集成

```yaml
context:
  enabled: true
  variable_selector: ["knowledge", "result"]
```

### 对话记忆

```yaml
memory:
  role_prefix:
    user: "用户"
    assistant: "助手"
  window:
    enabled: true
    size: 10  # 保留最近10轮对话
```

## 视觉能力

### 多模态输入

支持图像输入的模型（GPT-4V、Claude 3、Gemini Pro Vision）：

```yaml
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4-vision-preview
    vision:
      enabled: true
      configs:
        detail: high  # high, low, auto
    prompt_template:
    - role: user
      text: "请分析这张图片：{{#sys.files#}}"
  id: vision_llm
```

## 结构化输出

### JSON Schema

确保 LLM 输出符合指定的 JSON 格式：

```yaml
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    output_schema:
      type: object
      properties:
        title:
          type: string
          description: 文章标题
        summary:
          type: string
          description: 文章摘要
        keywords:
          type: array
          items:
            type: string
          description: 关键词列表
      required: ["title", "summary"]
  id: structured_llm
```

## 实际案例

### 案例1：简单问答

```yaml
- data:
    type: llm
    model:
      provider: openai
      name: gpt-3.5-turbo
      completion_params:
        temperature: 0.7
        max_tokens: 500
    prompt_template:
    - role: system
      text: "你是一个友好的AI助手"
    - role: user
      text: "{{#sys.query#}}"
  id: qa_llm
```

### 案例2：RAG 问答

```yaml
- data:
    type: llm
    model:
      provider: anthropic
      name: claude-3-opus
      completion_params:
        temperature: 0.3
        max_tokens: 2000
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

        用户问题：{{#sys.query#}}
    context:
      enabled: true
      variable_selector: ["knowledge", "result"]
  id: rag_llm
```

### 案例3：代码生成

```yaml
- data:
    type: llm
    model:
      provider: deepseek
      name: deepseek-coder
      completion_params:
        temperature: 0.2
        max_tokens: 4000
    prompt_template:
    - role: system
      text: |
        你是一个专业的代码生成助手。

        要求：
        1. 生成高质量、可运行的代码
        2. 添加必要的注释
        3. 遵循最佳实践
        4. 包含错误处理
    - role: user
      text: |
        需求：{{#start.requirement#}}

        编程语言：{{#start.language#}}

        请生成完整的代码实现。
  id: code_gen_llm
```

### 案例4：多轮对话

```yaml
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
      completion_params:
        temperature: 0.8
        max_tokens: 1000
    prompt_template:
    - role: system
      text: "你是一个有趣的对话伙伴"
    - role: user
      text: "{{#sys.query#}}"
    memory:
      role_prefix:
        user: "用户"
        assistant: "AI"
      window:
        enabled: true
        size: 20
  id: chat_llm
```

## 输出变量

LLM 节点输出以下变量：

```yaml
{{#llm.text#}}              # 生成的文本
{{#llm.usage#}}             # Token 使用统计
{{#llm.usage.prompt_tokens#}}      # 输入 Token 数
{{#llm.usage.completion_tokens#}}  # 输出 Token 数
{{#llm.usage.total_tokens#}}       # 总 Token 数
```

## 最佳实践

### 1. 提示词工程

- **明确角色**: 在系统消息中清晰定义 AI 的角色
- **提供示例**: 使用 few-shot 示例提升输出质量
- **结构化输出**: 要求特定格式（如 Markdown、JSON）
- **设置约束**: 明确输出长度、风格、语气等要求

### 2. 模型选择

- **简单任务**: 使用 GPT-3.5-turbo、Qwen-Turbo（成本低）
- **复杂推理**: 使用 GPT-4、Claude 3 Opus（质量高）
- **代码生成**: 使用 DeepSeek-Coder、GPT-4（专业性强）
- **多模态**: 使用 GPT-4V、Claude 3、Gemini Pro Vision

### 3. 参数调优

- **事实性任务**: temperature = 0.0-0.3
- **创意任务**: temperature = 0.7-1.0
- **平衡任务**: temperature = 0.5-0.7

### 4. 成本优化

- 使用 `max_tokens` 限制输出长度
- 对简单任务使用轻量模型
- 启用缓存机制（如果支持）
- 批量处理减少 API 调用

### 5. 错误处理

- 设置合理的超时时间
- 配置重试策略
- 提供降级方案（使用备用模型）

## 性能优化

### 1. 提示词优化

- 精简提示词，减少 Token 消耗
- 使用变量引用避免重复内容
- 移除不必要的说明和示例

### 2. 并行处理

- 对独立的 LLM 调用使用并行分支
- 避免串行依赖

### 3. 缓存策略

- 对相同输入启用缓存
- 使用知识库减少重复查询

## 常见问题

### Q: 如何选择合适的模型？

A: 根据任务复杂度、成本预算和响应速度综合考虑：
- 简单任务：GPT-3.5-turbo、Qwen-Turbo
- 复杂任务：GPT-4、Claude 3 Opus
- 代码任务：DeepSeek-Coder、GPT-4
- 多模态：GPT-4V、Claude 3

### Q: 如何减少 Token 消耗？

A:
1. 精简提示词
2. 设置合理的 `max_tokens`
3. 使用轻量模型处理简单任务
4. 启用缓存机制

### Q: 如何提升输出质量？

A:
1. 优化提示词（明确角色、提供示例）
2. 使用结构化输出
3. 调整 temperature 参数
4. 集成知识库提供上下文

### Q: 如何处理 LLM 超时？

A:
1. 设置合理的超时时间
2. 配置重试策略
3. 使用条件分支处理失败情况
4. 提供降级方案

## 相关节点

- [Agent 节点](./agent.md) - 智能代理，自主调用工具
- [Knowledge Retrieval 节点](./knowledge-retrieval.md) - 知识库检索
- [Code 节点](./code.md) - 数据预处理和后处理

## 返回

[工作流 SKILL](../SKILL.md)

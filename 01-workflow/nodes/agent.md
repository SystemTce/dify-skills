# 智能代理节点详解

## 概述

智能代理 (Agent) 节点使 LLM 能够自主控制工具，动态推理使用哪些工具以及何时使用，而不是遵循预定的顺序。

## 核心功能

- 自主工具调用和决策
- 动态推理和规划
- 多步骤任务执行
- 工具链编排

## 策略选项

### Function Calling (函数调用)

利用 LLM 的原生能力，最适合 GPT-4 和 Claude 3.5 等模型。

**优势**:
- 原生支持，性能更好
- 更准确的工具选择
- 更低的 Token 消耗

**配置示例**:
```yaml
- data:
    type: agent
    agent_strategy_name: function_calling
    agent_parameters:
      model:
        value:
          provider: openai
          model: gpt-4
      instruction:
        value: "根据用户需求调用合适的工具"
      query:
        value: "{{#sys.query#}}"
      tools:
        value:
        - enabled: true
          provider_name: google
          tool_name: search
          type: builtin
  id: agent
```

### ReAct (推理和行动)

使用结构化提示和显式推理步骤，适合不支持原生函数调用的模型。

**优势**:
- 兼容更多模型
- 推理过程可见
- 更容易调试

**配置示例**:
```yaml
- data:
    type: agent
    agent_strategy_name: react
    agent_parameters:
      model:
        value:
          provider: anthropic
          model: claude-3-sonnet
      instruction:
        value: |
          你是一个智能助手。
          思考：分析问题
          行动：选择工具
          观察：查看结果
          重复直到完成任务
      query:
        value: "{{#sys.query#}}"
      tools:
        value:
        - enabled: true
          provider_name: weather
          tool_name: get_weather
          type: builtin
  id: react_agent
```

## 关键配置

### 模型选择

选择支持所选策略的模型：

**Function Calling 推荐**:
- GPT-4, GPT-4 Turbo
- Claude 3.5 Sonnet, Claude 3 Opus
- Gemini Pro

**ReAct 推荐**:
- 任何支持长上下文的模型
- Claude 系列
- 开源模型（Llama, Qwen 等）

### 工具配置

```yaml
tools:
  value:
  # 内置工具
  - enabled: true
    provider_name: google
    tool_name: search
    type: builtin

  # 自定义工具
  - enabled: true
    provider_name: custom
    tool_name: database_query
    type: custom

  # 工作流工具
  - enabled: true
    provider_name: workflow
    tool_name: data_processor
    type: workflow
```

### 指令配置

使用自然语言和 Jinja2 语法：

```yaml
instruction:
  value: |
    你是一个专业的研究助手。

    任务：{{#start.task#}}

    可用工具：
    - search: 搜索互联网信息
    - calculator: 执行数学计算
    - database: 查询数据库

    要求：
    1. 仔细分析用户需求
    2. 选择合适的工具
    3. 验证结果的准确性
    4. 提供清晰的解释
```

### 查询字段

指定代理应处理的用户输入或任务：

```yaml
query:
  value: "{{#sys.query#}}"
  # 或引用其他节点
  value: "{{#start.user_request#}}"
```

## 执行控制

### 最大迭代次数

防止无限循环：

```yaml
agent_parameters:
  max_iterations:
    value: 10  # 简单任务: 3-5, 复杂任务: 10-15
```

**建议值**:
- 简单查询：3-5 次
- 中等复杂度：5-10 次
- 复杂研究：10-15 次

### 记忆管理

使用 TokenBufferMemory 添加上下文：

```yaml
agent_parameters:
  memory:
    enabled: true
    type: token_buffer
    max_tokens: 2000
```

**注意**: 启用记忆会增加 Token 成本，但能实现对话连续性。

### 工具参数模式

**自动生成** (推荐):
```yaml
tools:
  value:
  - tool_name: search
    parameters:
      query:
        mode: auto  # 由代理填充
```

**手动输入**:
```yaml
tools:
  value:
  - tool_name: api_call
    parameters:
      api_key:
        mode: manual
        value: "{{#start.api_key#}}"  # 永久配置值
```

## 输出变量

```yaml
{{#agent.text#}}              # 最终答案
{{#agent.tool_calls#}}        # 工具调用列表
{{#agent.reasoning#}}         # 推理轨迹（ReAct）
{{#agent.iterations#}}        # 迭代次数
{{#agent.success#}}           # 执行状态
{{#agent.logs#}}              # 结构化日志
```

## 实际案例

### 案例1：研究助手

```yaml
- data:
    type: agent
    agent_strategy_name: function_calling
    agent_parameters:
      model:
        value:
          provider: openai
          model: gpt-4
      instruction:
        value: |
          你是一个研究助手。

          任务：深入研究用户提出的问题

          步骤：
          1. 使用搜索工具收集信息
          2. 使用计算器进行数据分析
          3. 综合信息提供答案
      query:
        value: "{{#sys.query#}}"
      max_iterations:
        value: 10
      tools:
        value:
        - enabled: true
          provider_name: google
          tool_name: search
          type: builtin
        - enabled: true
          provider_name: calculator
          tool_name: calculate
          type: builtin
  id: research_agent
```

### 案例2：故障排查

```yaml
- data:
    type: agent
    agent_strategy_name: react
    agent_parameters:
      model:
        value:
          provider: anthropic
          model: claude-3-opus
      instruction:
        value: |
          你是一个系统故障排查专家。

          问题：{{#start.issue#}}

          可用工具：
          - check_logs: 检查系统日志
          - check_status: 检查服务状态
          - run_diagnostic: 运行诊断测试

          流程：
          1. 思考：分析可能的原因
          2. 行动：使用工具收集信息
          3. 观察：分析工具输出
          4. 重复直到找到根本原因
      query:
        value: "诊断并解决系统问题"
      max_iterations:
        value: 15
      tools:
        value:
        - enabled: true
          provider_name: monitoring
          tool_name: check_logs
          type: custom
        - enabled: true
          provider_name: monitoring
          tool_name: check_status
          type: custom
  id: troubleshoot_agent
```

### 案例3：多步骤数据处理

```yaml
- data:
    type: agent
    agent_strategy_name: function_calling
    agent_parameters:
      model:
        value:
          provider: openai
          model: gpt-4-turbo
      instruction:
        value: |
          你是一个数据处理专家。

          数据源：{{#start.data_source#}}
          处理要求：{{#start.requirements#}}

          步骤：
          1. 从数据库提取数据
          2. 清洗和转换数据
          3. 执行分析
          4. 生成报告
      query:
        value: "处理数据并生成报告"
      max_iterations:
        value: 8
      memory:
        enabled: true
        max_tokens: 3000
      tools:
        value:
        - enabled: true
          provider_name: database
          tool_name: query
          type: custom
        - enabled: true
          provider_name: data_processing
          tool_name: transform
          type: workflow
        - enabled: true
          provider_name: analytics
          tool_name: analyze
          type: custom
  id: data_agent
```

## 常见使用场景

### 研究和分析
- 多源信息收集
- 数据分析和可视化
- 市场研究
- 竞品分析

### 故障排查
- 系统诊断
- 日志分析
- 性能调优
- 问题定位

### 多步骤数据处理
- ETL 流程
- 数据清洗
- 报告生成
- 数据集成

### 动态 API 集成
- 根据响应调整后续请求
- 多 API 协调
- 错误处理和重试
- 数据聚合

## 最佳实践

### 1. 指令设计
- 清晰定义代理的角色和目标
- 提供工具使用指南
- 设置明确的成功标准
- 包含错误处理指导

### 2. 工具选择
- 只添加必要的工具
- 提供清晰的工具描述
- 设置合理的参数验证
- 测试工具的可靠性

### 3. 迭代控制
- 根据任务复杂度设置最大迭代
- 监控迭代次数
- 实现提前退出条件
- 记录异常情况

### 4. 成本优化
- 选择合适的模型
- 限制记忆窗口大小
- 优化工具调用次数
- 使用缓存减少重复调用

### 5. 错误处理
- 实现工具调用失败处理
- 设置超时机制
- 提供降级方案
- 记录详细日志

## 性能优化

### 模型选择
- 简单任务：GPT-3.5-turbo
- 复杂推理：GPT-4, Claude 3 Opus
- 成本敏感：使用较小模型 + ReAct

### 工具优化
- 减少工具数量
- 优化工具响应时间
- 实现工具结果缓存
- 并行调用独立工具

### 提示词优化
- 精简指令
- 提供清晰的示例
- 使用结构化格式
- 避免冗余信息

## 常见问题

### Q: Function Calling 和 ReAct 如何选择？

A:
- 支持函数调用的模型 → Function Calling
- 需要可见推理过程 → ReAct
- 成本敏感 → Function Calling
- 调试需求 → ReAct

### Q: 如何控制代理的执行成本？

A:
1. 选择合适的模型
2. 限制最大迭代次数
3. 减少工具数量
4. 优化提示词长度
5. 限制记忆窗口大小

### Q: 代理陷入循环怎么办？

A:
1. 降低最大迭代次数
2. 优化指令，提供更清晰的目标
3. 检查工具输出是否有用
4. 添加提前退出条件

### Q: 如何提升代理的准确性？

A:
1. 使用更强大的模型
2. 提供详细的指令和示例
3. 优化工具描述
4. 添加验证步骤
5. 启用记忆功能

## 相关节点

- [LLM 节点](./llm.md) - 基础语言模型调用
- [工具节点](./tools.md) - 工具集成
- [Code 节点](./code.md) - 自定义逻辑

## 返回

[工作流 SKILL](../SKILL.md)

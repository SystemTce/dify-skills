# DSL 语法参考

> Dify 工作流 DSL 完整语法参考

## 文件结构

```yaml
# 工作流 DSL 文件结构
version: "0.0.1"           # DSL 版本
graph:                      # 工作流图
  nodes: []                # 节点列表
  edges: []                # 边列表
  variables: []            # 变量定义
```

## 版本

| 版本 | Dify 版本 | 说明 |
|------|----------|------|
| 0.0.1 | 0.3.0+ | 初始版本 |
| 0.0.2 | 0.6.0+ | 添加新节点类型 |
| 0.0.3 | 0.9.0+ | 支持并行执行 |

## 节点定义

### 节点结构

```yaml
- id: node_1              # 节点唯一 ID
  data:                   # 节点数据
    type: llm             # 节点类型
    # ... 类型特定配置
  position:               # 可视化位置
    x: 0
    y: 0
```

### Start 节点

```yaml
- id: start
  data:
    type: start
    variables:
      - name: query
        type: text
        required: true
        default: ""
      - name: file
        type: file
        required: false
```

### LLM 节点

```yaml
- id: llm_1
  data:
    type: llm
    model:
      provider: openai
      name: gpt-4
      mode: chat
      completion_params:
        temperature: 0.7
        max_tokens: 2000
        top_p: 0.9
    prompt_template:
      - role: system
        text: "你是一个有帮助的助手"
      - role: user
        text: "{{#sys.query#}}"
    context:
      enabled: true
      variable_selector: ["knowledge", "result"]
    vision:
      enabled: false
```

### Agent 节点

```yaml
- id: agent_1
  data:
    type: agent
    agent_strategy_name: function_calling
    agent_parameters:
      instruction:
        value: "根据用户问题调用合适的工具"
      model:
        value:
          provider: openai
          model: gpt-4-turbo
      query:
        value: "{{#sys.query#}}"
      tools:
        value:
          - enabled: true
            provider_name: wikipedia
            tool_name: wiki_search
            type: builtin
```

### HTTP Request 节点

```yaml
- id: http_1
  data:
    type: http-request
    method: POST
    url: https://api.example.com/search
    headers:
      Content-Type: application/json
      Authorization: "Bearer {{#env.API_KEY#}}"
    body: |
      {
        "query": "{{#sys.query#}}",
        "limit": 5
      }
    body_type: json
    timeout: 30
```

### Code 节点

```yaml
- id: code_1
  data:
    type: code
    code_type: python
    code: |
      # 处理数据
      result = {
          "processed": input_text.upper(),
          "length": len(input_text)
      }
    inputs:
      - name: input_text
        type: string
        value_selector: ["start", "query"]
    outputs:
      - name: result
        type: object
```

### If-Else 节点

```yaml
- id: condition_1
  data:
    type: if-else
    conditions:
      - comparison_operator: contains
        value: "{{#sys.query#}}"
        sub_condition: null
        variable: query
        variable_type: string
    logical_operator: and
```

### Iteration 节点

```yaml
- id: iteration_1
  data:
    type: iteration
    iterator_variable_name: items
    output_schema:
      - name: item
        type: string
```

### Answer 节点

```yaml
- id: answer_1
  data:
    type: answer
    answer: "{{#llm_1.text#}}"
    from_variable_selector:
      - llm_1
      - text
```

## 边定义

### 基本边

```yaml
edges:
  - source: start
    target: llm_1
```

### 带条件边

```yaml
edges:
  - source: condition_1
    target: llm_1
    source_handle: true  # 条件为 true 时的分支
  - source: condition_1
    target: llm_2
    source_handle: false  # 条件为 false 时的分支
```

## 变量定义

### 变量类型

```yaml
variables:
  - name: user_input        # 变量名
    type: text             # 类型: text, number, boolean, select, file, json
    required: true         # 是否必填
    default: ""            # 默认值
    options: []           # 选项 (select 类型)
    max_length: 1000      # 最大长度 (text 类型)
    precision: 2          # 精度 (number 类型)
```

### 变量引用

```yaml
# 引用变量
{{#变量名#}}

# 引用节点输出
{{#llm_1.text#}}

# 引用环境变量
{{#env.API_KEY#}}

# 引用系统变量
{{#sys.query#}}
{{#sys.files#}}
{{#sys.conversation_id#}}
```

## 完整示例

```yaml
version: "0.0.3"
graph:
  nodes:
    - id: start
      data:
        type: start
        variables:
          - name: query
            type: text
            required: true

    - id: classify
      data:
        type: question-classifier
        instruction: "判断用户问题是技术问题还是一般问题"
        categories:
          - name: technical
            description: 技术问题
          - name: general
            description: 一般问题

    - id: technical_llm
      data:
        type: llm
        model:
          provider: openai
          name: gpt-4
        prompt_template:
          - role: system
            text: "你是一个技术专家"
          - role: user
            text: "{{#start.query#}}"

    - id: general_llm
      data:
        type: llm
        model:
          provider: openai
          name: gpt-3.5-turbo
        prompt_template:
          - role: system
            text: "你是一个通用助手"
          - role: user
            text: "{{#start.query#}}"

    - id: answer
      data:
        type: answer

  edges:
    - source: start
      target: classify
    - source: classify
      target: technical_llm
      source_handle: technical
    - source: classify
      target: general_llm
      source_handle: general
    - source: technical_llm
      target: answer
    - source: general_llm
      target: answer
```

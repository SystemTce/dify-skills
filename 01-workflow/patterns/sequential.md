# 顺序管道模式 (Sequential Pipeline)

## 概述

顺序管道模式是最基础的工作流设计模式，节点按照线性顺序依次执行，适合简单的单向处理流程。

## 模式结构

```
Start → Processing Node(s) → Answer
```

## 适用场景

- 简单的问答应用
- 文本生成任务
- 数据转换流程
- 单步骤处理

## 优势

- 结构简单，易于理解
- 调试方便
- 执行路径明确
- 适合快速原型

## 劣势

- 缺乏灵活性
- 无法处理复杂逻辑
- 不支持条件分支
- 难以扩展

## 基础示例

### 示例1：简单问答

**流程**: Start → LLM → Answer

```yaml
workflow:
  nodes:
  # 1. Start 节点
  - data:
      type: start
      variables:
      - label: 用户问题
        type: text-input
        variable: question
    id: start

  # 2. LLM 节点
  - data:
      type: llm
      model:
        provider: openai
        name: gpt-3.5-turbo
      prompt_template:
      - role: system
        text: "你是一个友好的AI助手"
      - role: user
        text: "{{#start.question#}}"
    id: llm

  # 3. Answer 节点
  - data:
      type: answer
      answer: "{{#llm.text#}}"
    id: answer

  edges:
  - source: start
    target: llm
  - source: llm
    target: answer
```

### 示例2：文本处理管道

**流程**: Start → Code (清洗) → LLM (分析) → Code (格式化) → Answer

```yaml
workflow:
  nodes:
  # 1. Start
  - data:
      type: start
      variables:
      - label: 原始文本
        type: paragraph
        variable: raw_text
    id: start

  # 2. Code - 文本清洗
  - data:
      type: code
      code_language: python3
      code: |
        def main(text: str) -> dict:
            # 清洗文本
            cleaned = text.strip().lower()
            # 移除多余空格
            cleaned = ' '.join(cleaned.split())
            return {"cleaned_text": cleaned}
      variables:
      - variable: text
        value_selector: ["start", "raw_text"]
    id: clean

  # 3. LLM - 文本分析
  - data:
      type: llm
      model:
        provider: openai
        name: gpt-4
      prompt_template:
      - role: user
        text: |
          请分析以下文本的主题和情感：
          {{#clean.cleaned_text#}}

          输出格式：
          主题：[主题]
          情感：[正面/负面/中性]
          关键词：[关键词列表]
    id: analyze

  # 4. Code - 格式化输出
  - data:
      type: code
      code: |
        def main(analysis: str) -> dict:
            # 解析 LLM 输出
            lines = analysis.strip().split('\n')
            result = {
                "topic": lines[0].split('：')[1] if len(lines) > 0 else "",
                "sentiment": lines[1].split('：')[1] if len(lines) > 1 else "",
                "keywords": lines[2].split('：')[1] if len(lines) > 2 else ""
            }
            return result
      variables:
      - variable: analysis
        value_selector: ["analyze", "text"]
    id: format

  # 5. Answer
  - data:
      type: answer
      answer: |
        分析结果：
        主题：{{#format.topic#}}
        情感：{{#format.sentiment#}}
        关键词：{{#format.keywords#}}
    id: answer

  edges:
  - source: start
    target: clean
  - source: clean
    target: analyze
  - source: analyze
    target: format
  - source: format
    target: answer
```

### 示例3：翻译管道

**流程**: Start → LLM (翻译) → Code (验证) → Answer

```yaml
workflow:
  nodes:
  - data:
      type: start
      variables:
      - label: 原文
        type: paragraph
        variable: source_text
      - label: 目标语言
        type: select
        variable: target_lang
        options: ["英语", "日语", "法语"]
    id: start

  - data:
      type: llm
      model:
        provider: openai
        name: gpt-4
      prompt_template:
      - role: system
        text: "你是一个专业的翻译助手"
      - role: user
        text: |
          请将以下文本翻译成{{#start.target_lang#}}：
          {{#start.source_text#}}

          要求：
          1. 保持原文的语气和风格
          2. 确保翻译准确流畅
          3. 只输出翻译结果，不要添加说明
    id: translate

  - data:
      type: code
      code: |
        def main(translation: str, source: str) -> dict:
            # 验证翻译结果
            is_valid = len(translation) > 0 and translation != source

            return {
                "translation": translation,
                "is_valid": is_valid,
                "char_count": len(translation)
            }
      variables:
      - variable: translation
        value_selector: ["translate", "text"]
      - variable: source
        value_selector: ["start", "source_text"]
    id: validate

  - data:
      type: answer
      answer: |
        翻译结果：
        {{#validate.translation#}}

        字符数：{{#validate.char_count#}}
    id: answer
```

## 高级示例

### 示例4：多步骤数据处理

**流程**: Start → HTTP (获取数据) → Code (解析) → LLM (分析) → Code (生成报告) → Answer

```yaml
workflow:
  nodes:
  - data:
      type: start
      variables:
      - label: API URL
        type: text-input
        variable: api_url
    id: start

  - data:
      type: http-request
      method: GET
      url: "{{#start.api_url#}}"
      timeout: 30
    id: fetch

  - data:
      type: code
      code: |
        import json
        def main(response: str) -> dict:
            data = json.loads(response)
            # 提取关键信息
            items = data.get("items", [])
            summary = {
                "total": len(items),
                "active": sum(1 for i in items if i.get("status") == "active"),
                "items": items[:10]  # 只取前10项
            }
            return summary
      variables:
      - variable: response
        value_selector: ["fetch", "body"]
    id: parse

  - data:
      type: llm
      model:
        provider: openai
        name: gpt-4
      prompt_template:
      - role: user
        text: |
          请分析以下数据并生成简要报告：
          总数：{{#parse.total#}}
          活跃数：{{#parse.active#}}

          请提供：
          1. 数据概况
          2. 关键发现
          3. 建议
    id: analyze

  - data:
      type: code
      code: |
        def main(analysis: str, total: int, active: int) -> dict:
            report = f"""
            # 数据分析报告

            ## 统计数据
            - 总数：{total}
            - 活跃数：{active}
            - 活跃率：{active/total*100:.1f}%

            ## AI 分析
            {analysis}
            """
            return {"report": report}
      variables:
      - variable: analysis
        value_selector: ["analyze", "text"]
      - variable: total
        value_selector: ["parse", "total"]
      - variable: active
        value_selector: ["parse", "active"]
    id: generate_report

  - data:
      type: answer
      answer: "{{#generate_report.report#}}"
    id: answer
```

## 性能优化

### 1. 减少节点数量

合并可以合并的处理步骤：

```python
# 不好：多个 Code 节点
Code 1: 清洗数据
Code 2: 验证数据
Code 3: 转换数据

# 好：单个 Code 节点
Code: 清洗 + 验证 + 转换
```

### 2. 优化 LLM 调用

- 使用合适的模型（简单任务用轻量模型）
- 精简提示词
- 设置合理的 `max_tokens`

### 3. 缓存策略

对相同输入启用缓存（如果支持）。

## 最佳实践

### 1. 节点命名

使用清晰的节点名称：

```yaml
# 不好
id: node1, node2, node3

# 好
id: clean_text, analyze_sentiment, format_output
```

### 2. 错误处理

在关键节点添加验证：

```python
def main(data: str) -> dict:
    if not data:
        return {"error": "数据为空"}

    # 处理逻辑
    result = process(data)

    return {"result": result}
```

### 3. 数据验证

在管道开始时验证输入：

```python
def main(input_text: str) -> dict:
    # 验证
    if len(input_text) > 10000:
        return {"error": "文本过长"}

    return {"validated_text": input_text}
```

### 4. 输出格式化

在管道结束时格式化输出：

```python
def main(result: dict) -> dict:
    formatted = f"""
    结果：{result['data']}
    处理时间：{result['time']}
    状态：成功
    """
    return {"output": formatted}
```

## 何时使用

### 适合使用

- 简单的单向处理流程
- 快速原型开发
- 学习和测试
- 明确的线性逻辑

### 不适合使用

- 需要条件分支
- 需要循环处理
- 复杂的业务逻辑
- 需要错误恢复

## 扩展方向

### 升级到条件分支模式

添加 If-Else 节点处理不同情况：

```
Start → LLM → If-Else → [分支A / 分支B] → Answer
```

### 升级到迭代处理模式

添加 Iteration 节点处理批量数据：

```
Start → Code → Iteration → LLM → Answer
```

## 相关模式

- [条件分支模式](./conditional.md) - 添加条件判断
- [迭代处理模式](./iterative.md) - 批量处理
- [多源聚合模式](./aggregation.md) - 整合多个数据源

## 返回

[工作流 SKILL](../SKILL.md)

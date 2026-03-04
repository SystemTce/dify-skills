# Iteration 节点详解

## 概述

Iteration 节点用于迭代处理数组数据，对每个元素执行相同的处理逻辑，支持并行执行和错误处理。

## 核心功能

- 批量数据处理
- 并行执行支持
- 错误处理策略
- 结果聚合

## 配置结构

```yaml
- data:
    type: iteration
    title: 批量处理
    iterator_selector: ["start", "items"]
    output_selector: ["iteration_inner", "output"]
    parallel_nums: 3
    error_handle_mode: continue
  id: iteration
```

## 配置参数

### iterator_selector (迭代数据源)

指定要迭代的数组数据：

```yaml
iterator_selector: ["start", "items"]  # 来自 Start 节点的 items 数组
iterator_selector: ["code", "list"]    # 来自 Code 节点的 list 数组
```

### output_selector (输出选择器)

指定迭代内部节点的输出：

```yaml
output_selector: ["llm_inner", "text"]  # 收集 LLM 节点的文本输出
```

### parallel_nums (并行数量)

控制并行执行的数量：

```yaml
parallel_nums: 1   # 串行执行
parallel_nums: 3   # 3个并行
parallel_nums: 10  # 10个并行
```

**建议**:
- 轻量任务：5-10
- LLM 调用：2-5
- 重量任务：1-3

### error_handle_mode (错误处理模式)

**continue**: 继续执行，跳过失败项

```yaml
error_handle_mode: continue
```

**stop**: 遇到错误立即停止

```yaml
error_handle_mode: stop
```

## 迭代变量

### 当前项 (item)

访问当前迭代的元素：

```yaml
{{#item#}}              # 当前元素
{{#item.name#}}         # 当前元素的 name 字段
{{#item.value#}}        # 当前元素的 value 字段
```

### 当前索引 (index)

访问当前迭代的索引（从 0 开始）：

```yaml
{{#index#}}             # 当前索引
```

## 实际案例

### 案例1：批量文本处理

```yaml
# 迭代节点配置
- data:
    type: iteration
    iterator_selector: ["start", "texts"]
    output_selector: ["llm_inner", "text"]
    parallel_nums: 5
    error_handle_mode: continue
  id: batch_process

# 迭代内部的 LLM 节点
- data:
    type: llm
    model:
      provider: openai
      name: gpt-3.5-turbo
    prompt_template:
    - role: user
      text: "请总结以下文本：{{#item#}}"
  id: llm_inner
  parent_id: batch_process  # 标记为迭代内部节点
```

### 案例2：批量数据转换

```yaml
# 迭代节点
- data:
    type: iteration
    iterator_selector: ["start", "users"]
    output_selector: ["code_inner", "result"]
    parallel_nums: 10
    error_handle_mode: continue
  id: transform_users

# 迭代内部的 Code 节点
- data:
    type: code
    code_language: python3
    code: |
      def main(user: dict, index: int) -> dict:
          return {
              "id": user["id"],
              "name": user["name"].upper(),
              "index": index,
              "processed": True
          }
    variables:
    - variable: user
      value_selector: ["item"]
    - variable: index
      value_selector: ["index"]
  id: code_inner
  parent_id: transform_users
```

### 案例3：批量 API 调用

```yaml
# 迭代节点
- data:
    type: iteration
    iterator_selector: ["code", "urls"]
    output_selector: ["http_inner", "body"]
    parallel_nums: 3
    error_handle_mode: continue
  id: fetch_data

# 迭代内部的 HTTP 节点
- data:
    type: http-request
    method: GET
    url: "{{#item#}}"
    timeout: 30
  id: http_inner
  parent_id: fetch_data
```

### 案例4：多步骤迭代处理

```yaml
# 迭代节点
- data:
    type: iteration
    iterator_selector: ["start", "documents"]
    output_selector: ["answer_inner", "text"]
    parallel_nums: 2
    error_handle_mode: continue
  id: process_docs

# 内部流程：提取 → LLM 分析 → 格式化
# 1. 提取文本
- data:
    type: code
    code: |
      def main(doc: dict) -> dict:
          return {"text": doc.get("content", "")}
    variables:
    - variable: doc
      value_selector: ["item"]
  id: extract_inner
  parent_id: process_docs

# 2. LLM 分析
- data:
    type: llm
    prompt_template:
    - role: user
      text: "分析文档：{{#extract_inner.text#}}"
  id: llm_inner
  parent_id: process_docs

# 3. 格式化输出
- data:
    type: code
    code: |
      def main(analysis: str, index: int) -> dict:
          return {
              "index": index,
              "analysis": analysis,
              "timestamp": datetime.now().isoformat()
          }
    variables:
    - variable: analysis
      value_selector: ["llm_inner", "text"]
    - variable: index
      value_selector: ["index"]
  id: answer_inner
  parent_id: process_docs
```

## 输出结果

### 结果数组

迭代节点输出一个数组，包含所有迭代的结果：

```yaml
{{#iteration.output#}}  # 结果数组
```

### 访问结果

在后续节点中访问迭代结果：

```yaml
# 在 Code 节点中处理结果
def main(results: list) -> dict:
    # results 是迭代的输出数组
    success_count = len([r for r in results if r.get("success")])

    return {
        "total": len(results),
        "success": success_count,
        "failed": len(results) - success_count
    }
```

## 性能优化

### 1. 并行数量调优

```yaml
# 根据任务类型调整
parallel_nums: 1   # CPU 密集型任务
parallel_nums: 5   # LLM API 调用
parallel_nums: 10  # 轻量级处理
```

### 2. 批量大小控制

```python
# 在迭代前分批
def main(items: list) -> dict:
    batch_size = 100
    batches = [items[i:i+batch_size] for i in range(0, len(items), batch_size)]
    return {"batches": batches}
```

### 3. 错误处理

```yaml
# 使用 continue 模式避免单个失败影响整体
error_handle_mode: continue
```

## 最佳实践

### 1. 数据准备

```python
# 在迭代前准备数据
def main(raw_data: list) -> dict:
    # 清洗和验证
    cleaned = [item for item in raw_data if item.get("id")]

    # 添加索引或元数据
    prepared = [
        {"index": i, "data": item}
        for i, item in enumerate(cleaned)
    ]

    return {"items": prepared}
```

### 2. 结果聚合

```python
# 在迭代后聚合结果
def main(results: list) -> dict:
    # 统计
    total = len(results)
    success = sum(1 for r in results if r.get("success"))

    # 提取数据
    data = [r.get("data") for r in results if r.get("data")]

    return {
        "summary": {
            "total": total,
            "success": success,
            "failed": total - success
        },
        "data": data
    }
```

### 3. 错误记录

```python
# 记录失败项
def main(item: dict, index: int) -> dict:
    try:
        result = process(item)
        return {
            "success": True,
            "index": index,
            "data": result
        }
    except Exception as e:
        return {
            "success": False,
            "index": index,
            "error": str(e),
            "item": item
        }
```

### 4. 进度跟踪

```python
# 在迭代内部输出进度
def main(item: dict, index: int, total: int) -> dict:
    result = process(item)

    return {
        "progress": f"{index + 1}/{total}",
        "percentage": (index + 1) / total * 100,
        "data": result
    }
```

## 常见问题

### Q: 如何获取迭代总数？

A: 在迭代前使用 Code 节点计算：
```python
def main(items: list) -> dict:
    return {"items": items, "total": len(items)}
```

### Q: 如何处理嵌套数组？

A: 使用 Code 节点展平数组，或使用嵌套迭代。

### Q: 迭代失败如何重试？

A: 在迭代内部使用 If-Else 节点检查结果，失败时重新处理。

### Q: 如何限制迭代数量？

A: 在迭代前使用 Code 节点切片数组：
```python
def main(items: list, limit: int) -> dict:
    return {"items": items[:limit]}
```

## 性能考虑

### 1. 内存使用

- 大数组会占用较多内存
- 建议单次迭代不超过 1000 项
- 超大数据集分批处理

### 2. 执行时间

- 串行执行：时间 = 单项时间 × 数量
- 并行执行：时间 ≈ 单项时间 × (数量 / 并行数)

### 3. API 限流

- 注意 API 调用频率限制
- 使用合理的 `parallel_nums`
- 添加重试逻辑

## 相关节点

- [Code 节点](./code.md) - 数据准备和结果聚合
- [LLM 节点](./llm.md) - 批量文本处理
- [HTTP Request 节点](./http-request.md) - 批量 API 调用

## 返回

[工作流 SKILL](../SKILL.md)

# If-Else 节点详解

## 概述

If-Else 节点用于实现条件分支逻辑，根据条件判断结果执行不同的处理路径。

## 核心功能

- 条件判断和分支控制
- 支持多个条件组合
- 支持多分支输出
- 灵活的比较运算符

## 配置结构

```yaml
- data:
    type: if-else
    title: 条件判断
    conditions:
    - variable_selector: ["llm", "text"]
      comparison_operator: contains
      value: "关键词"
    logical_operator: and
  id: condition
  position:
    x: 300
    y: 200
```

## 比较运算符

### 字符串运算符

- `contains`: 包含
- `not contains`: 不包含
- `start with`: 以...开头
- `end with`: 以...结尾
- `is`: 等于
- `is not`: 不等于
- `empty`: 为空
- `not empty`: 不为空

### 数字运算符

- `=`: 等于
- `≠`: 不等于
- `>`: 大于
- `<`: 小于
- `≥`: 大于等于
- `≤`: 小于等于

### 布尔运算符

- `is`: 等于 true/false
- `is not`: 不等于 true/false

## 逻辑运算符

### AND (与)

所有条件都必须满足：

```yaml
conditions:
- variable_selector: ["llm", "text"]
  comparison_operator: contains
  value: "Python"
- variable_selector: ["start", "level"]
  comparison_operator: ">"
  value: 5
logical_operator: and
```

### OR (或)

任一条件满足即可：

```yaml
conditions:
- variable_selector: ["llm", "text"]
  comparison_operator: contains
  value: "Python"
- variable_selector: ["llm", "text"]
  comparison_operator: contains
  value: "JavaScript"
logical_operator: or
```

## 实际案例

### 案例1：文本分类

```yaml
- data:
    type: if-else
    conditions:
    - variable_selector: ["llm", "text"]
      comparison_operator: contains
      value: "技术"
    logical_operator: and
  id: classify
```

**分支**:
- `true`: 技术类内容处理
- `false`: 其他类内容处理

### 案例2：数值范围判断

```yaml
- data:
    type: if-else
    conditions:
    - variable_selector: ["code", "score"]
      comparison_operator: ">="
      value: 80
    logical_operator: and
  id: score_check
```

**分支**:
- `true`: 高分处理
- `false`: 低分处理

### 案例3：多条件组合

```yaml
- data:
    type: if-else
    conditions:
    - variable_selector: ["start", "user_type"]
      comparison_operator: is
      value: "VIP"
    - variable_selector: ["code", "amount"]
      comparison_operator: ">"
      value: 1000
    logical_operator: and
  id: vip_check
```

**逻辑**: VIP 用户且金额大于 1000

### 案例4：空值检查

```yaml
- data:
    type: if-else
    conditions:
    - variable_selector: ["knowledge", "result"]
      comparison_operator: not empty
    logical_operator: and
  id: result_check
```

**用途**: 检查知识库是否返回结果

### 案例5：多分支判断

通过嵌套 If-Else 实现多分支：

```
If-Else 1 (score >= 90)
  ├─ true → 优秀
  └─ false → If-Else 2 (score >= 60)
      ├─ true → 及格
      └─ false → 不及格
```

## 分支连接

### True 分支

条件满足时执行的路径：

```yaml
edges:
- source: condition
  sourceHandle: "true"
  target: next_node_1
```

### False 分支

条件不满足时执行的路径：

```yaml
edges:
- source: condition
  sourceHandle: "false"
  target: next_node_2
```

### 自定义分支

可以添加多个自定义分支（通过分支 ID）：

```yaml
edges:
- source: condition
  sourceHandle: "branch-id-1"
  target: next_node_3
```

## 最佳实践

### 1. 条件设计

- 保持条件简单明确
- 避免过于复杂的逻辑组合
- 使用有意义的分支命名

### 2. 分支处理

- 确保所有分支都有后续处理
- 避免分支路径过长
- 考虑默认分支（false 分支）

### 3. 性能优化

- 将最常见的条件放在前面
- 避免不必要的嵌套
- 使用代码节点处理复杂逻辑

### 4. 错误处理

- 为空值情况提供处理分支
- 使用 `not empty` 检查数据有效性
- 提供友好的错误提示

## 常见模式

### 模式1：数据验证

```yaml
# 检查输入是否有效
conditions:
- variable_selector: ["start", "input"]
  comparison_operator: not empty
```

### 模式2：权限检查

```yaml
# 检查用户权限
conditions:
- variable_selector: ["start", "role"]
  comparison_operator: is
  value: "admin"
```

### 模式3：内容过滤

```yaml
# 过滤敏感内容
conditions:
- variable_selector: ["llm", "text"]
  comparison_operator: not contains
  value: "敏感词"
```

### 模式4：状态判断

```yaml
# 检查处理状态
conditions:
- variable_selector: ["code", "success"]
  comparison_operator: is
  value: true
```

## 常见问题

### Q: 如何实现多分支判断？

A: 使用嵌套的 If-Else 节点，或使用 Code 节点返回分支标识。

### Q: 可以比较两个变量吗？

A: 目前只支持变量与常量比较，变量间比较需要使用 Code 节点。

### Q: 如何处理复杂的逻辑？

A: 使用 Code 节点实现复杂逻辑，返回布尔值，然后用 If-Else 判断。

### Q: 分支可以合并吗？

A: 可以，多个分支可以连接到同一个后续节点。

## 相关节点

- [Code 节点](./code.md) - 实现复杂逻辑判断
- [LLM 节点](./llm.md) - 基于 LLM 输出进行判断
- [Answer 节点](./answer.md) - 不同分支的输出

## 返回

[工作流 SKILL](../SKILL.md)

# 循环节点详解

## 概述

循环节点实现迭代工作流，每个循环周期都基于前一次的结果构建，与独立的数组迭代不同。

## 核心功能

- 顺序处理，状态持久化
- 循环变量跨周期保持
- 退出条件控制
- 最大迭代限制

## 循环 vs 迭代

| 特性 | 循环节点 | 迭代节点 |
|------|---------|---------|
| 处理方式 | 顺序，有状态 | 独立，可并行 |
| 状态 | 跨周期持久化 | 每项独立 |
| 适用场景 | 渐进式优化 | 批量处理 |
| 依赖关系 | 后续依赖前次 | 项目间独立 |

## 配置结构

### 基础配置

```yaml
- data:
    type: loop
    loop_variables:
    - name: counter
      type: number
      initial_value: 0
    - name: result
      type: string
      initial_value: ""
    exit_condition: "{{#counter#}} >= 10"
    max_iterations: 20
  id: loop
```

## 循环变量

在整个执行过程中维护状态：

```yaml
loop_variables:
- name: num
  type: number
  initial_value: 0
- name: text
  type: string
  initial_value: ""
- name: data
  type: object
  initial_value: {}
```

## 终止控制

### 退出条件

```yaml
exit_condition: "{{#quality_score#}} > 0.9"
exit_condition: "{{#counter#}} >= {{#max_count#}}"
exit_condition: "{{#result.status#}} == 'success'"
```

### 最大迭代限制

防止无限循环的安全保护：

```yaml
max_iterations: 10  # 简单任务
max_iterations: 50  # 复杂任务
```

### 退出循环节点

立即终止循环：

```yaml
- data:
    type: exit-loop
    condition: "{{#check.is_complete#}}"
  id: exit_loop
```

## 实际案例

### 案例1：随机数生成（基础示例）

```yaml
# 循环节点
- data:
    type: loop
    loop_variables:
    - name: random_num
      type: number
      initial_value: 100
    exit_condition: "{{#random_num#}} < 50"
    max_iterations: 100
  id: loop

# 代码节点 - 生成随机数
- data:
    type: code
    code: |
      import random
      def main():
          return {"random_num": random.randint(1, 100)}
  id: generate_random

# 条件检查
- data:
    type: if-else
    conditions:
    - variable_selector: ["generate_random", "random_num"]
      comparison_operator: "<"
      value: 50
  id: check_condition

# 退出循环
- data:
    type: exit-loop
  id: exit_loop

# 模板节点 - 显示结果
- data:
    type: template
    template: "找到的数字：{{#generate_random.random_num#}}"
  id: show_result
```

### 案例2：迭代式诗歌优化

```yaml
# 循环节点
- data:
    type: loop
    loop_variables:
    - name: num
      type: number
      initial_value: 0
    - name: verse
      type: string
      initial_value: "初始诗句"
    exit_condition: "{{#num#}} >= 3"
    max_iterations: 5
  id: poetry_loop

# LLM 节点 - 改进诗句
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: system
      text: "你是一个诗歌创作专家"
    - role: user
      text: |
        当前诗句（第{{#num#}}次迭代）：
        {{#verse#}}

        请改进这首诗，使其更加优美。
  id: improve_poem

# 变量赋值 - 更新循环变量
- data:
    type: variable-assigner
    variables:
    - variable: num
      value: "{{#num#}} + 1"
    - variable: verse
      value: "{{#improve_poem.text#}}"
  id: update_variables
```

### 案例3：内容质量优化

```yaml
# 循环节点
- data:
    type: loop
    loop_variables:
    - name: iteration
      type: number
      initial_value: 0
    - name: content
      type: string
      initial_value: "{{#start.initial_content#}}"
    - name: quality_score
      type: number
      initial_value: 0
    exit_condition: "{{#quality_score#}} > 0.9"
    max_iterations: 10
  id: quality_loop

# LLM 节点 - 改进内容
- data:
    type: llm
    prompt_template:
    - role: user
      text: |
        当前内容：
        {{#content#}}

        当前质量分数：{{#quality_score#}}

        请改进内容以提高质量。
  id: improve_content

# LLM 节点 - 评估质量
- data:
    type: llm
    prompt_template:
    - role: user
      text: |
        评估以下内容的质量（0-1分）：
        {{#improve_content.text#}}

        返回JSON格式：{"score": 0.85}
    output_schema:
      type: object
      properties:
        score:
          type: number
  id: evaluate_quality

# 变量赋值 - 更新状态
- data:
    type: variable-assigner
    variables:
    - variable: iteration
      value: "{{#iteration#}} + 1"
    - variable: content
      value: "{{#improve_content.text#}}"
    - variable: quality_score
      value: "{{#evaluate_quality.score#}}"
  id: update_state
```

### 案例4：问题分解

```yaml
# 循环节点
- data:
    type: loop
    loop_variables:
    - name: step
      type: number
      initial_value: 1
    - name: current_problem
      type: string
      initial_value: "{{#start.problem#}}"
    - name: solutions
      type: array
      initial_value: []
    exit_condition: "{{#step#}} > 5"
    max_iterations: 10
  id: problem_solving_loop

# LLM 节点 - 分解问题
- data:
    type: llm
    prompt_template:
    - role: user
      text: |
        步骤 {{#step#}}：
        当前问题：{{#current_problem#}}

        请将问题分解为更小的子问题，并提供解决方案。
  id: decompose_problem

# 代码节点 - 处理结果
- data:
    type: code
    code: |
      def main(step, solutions, new_solution):
          solutions.append({
              "step": step,
              "solution": new_solution
          })
          return {
              "step": step + 1,
              "solutions": solutions,
              "is_complete": step >= 5
          }
  id: process_solution

# 条件检查
- data:
    type: if-else
    conditions:
    - variable_selector: ["process_solution", "is_complete"]
      comparison_operator: "equals"
      value: true
  id: check_complete

# 退出循环
- data:
    type: exit-loop
  id: exit_loop
```

## 常见应用

### 内容优化

通过重复的 LLM 审查周期优化内容：

```yaml
循环 → LLM 改进 → 质量评估 → 更新变量 → 检查退出条件
```

### 问题分解

将问题分解为迭代步骤：

```yaml
循环 → 分析问题 → 生成子步骤 → 更新状态 → 继续或退出
```

### 研究工作流

通过查询优化进行研究：

```yaml
循环 → 搜索 → 分析结果 → 优化查询 → 继续或完成
```

### 质量保证周期

质量保证循环：

```yaml
循环 → 生成内容 → 检查质量 → 修复问题 → 验证或重试
```

## 最佳实践

1. **定义明确的退出条件**
   - 使用可测量的指标
   - 设置合理的阈值
   - 避免模糊的条件

2. **设置适当的迭代限制**
   - 简单任务：3-5 次
   - 中等复杂度：5-10 次
   - 复杂任务：10-20 次

3. **高效管理状态**
   - 只保存必要的循环变量
   - 使用合适的数据类型
   - 避免存储大量数据

4. **包含进度跟踪**
   - 记录迭代次数
   - 跟踪改进指标
   - 记录中间结果

5. **实现安全退出**
   - 设置最大迭代限制
   - 添加超时机制
   - 处理异常情况

## 常见问题

### Q: 循环和迭代有什么区别？

A:
- **循环**：顺序处理，状态持久化，后续依赖前次结果
- **迭代**：独立处理，可并行，项目间无依赖

### Q: 如何避免无限循环？

A:
1. 设置明确的退出条件
2. 配置最大迭代限制
3. 添加超时机制
4. 监控循环执行

### Q: 循环变量如何更新？

A: 使用变量赋值器节点更新循环变量：
```yaml
- data:
    type: variable-assigner
    variables:
    - variable: counter
      value: "{{#counter#}} + 1"
```

### Q: 何时使用循环节点？

A: 适用于：
- 渐进式优化
- 迭代改进
- 问题分解
- 质量保证循环

不适用于：
- 批量数据处理（使用迭代节点）
- 独立任务执行
- 并行处理

## 相关节点

- [迭代节点](./iteration.md) - 独立项目处理
- [变量赋值器节点](./variable-assigner.md) - 更新循环变量
- [If-Else 节点](./if-else.md) - 条件控制

## 返回

[工作流 SKILL](../SKILL.md)

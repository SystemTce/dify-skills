# 变量聚合节点详解

## 概述

变量聚合节点将多个执行分支的变量合并为单一统一输出，解决分支问题，消除重复下游节点的需求。

## 核心功能

- 合并多分支输出
- 类型一致性检查
- 简化工作流结构
- 支持多种数据类型

## 核心价值

通过聚合分支输出，避免为每个分支重复处理逻辑。连接所有分支输出到此节点，下游只需引用单一变量。

## 关键特性

### 类型一致性
- 所有聚合变量必须相同数据类型
- 连接第一个变量后强制类型匹配
- 支持 String、Number、Object、Boolean、Array

### 输出选择
- 输出实际执行分支产生的值
- 未执行分支的值被忽略
- 保证只有一个有效输出

## 支持的数据类型

```yaml
- String: 文本字符串
- Number: 数值
- Object: JSON 对象
- Boolean: 布尔值
- Array: 数组
```

## 配置方法

### 基础聚合

```yaml
- data:
    type: variable-aggregator
    variables:
    - source: branch1
      variable: result
    - source: branch2
      variable: result
    - source: branch3
      variable: result
  id: aggregate_results
```

### 多组聚合（v0.6.10+）

```yaml
- data:
    type: variable-aggregator
    groups:
    - name: text_results
      variables:
      - source: path1
        variable: text
      - source: path2
        variable: text
    - name: score_results
      variables:
      - source: path1
        variable: score
      - source: path2
        variable: score
  id: multi_aggregate
```

## 输出变量

```yaml
{{#aggregate_results.output#}}      # 聚合后的值
{{#aggregate_results.source#}}      # 值的来源分支
```

## 实际案例

### 案例1：分类工作流

```yaml
# 问题分类
- data:
    type: question-classifier
    query: "{{#sys.query#}}"
    classes:
    - name: technical
      description: "技术问题"
    - name: billing
      description: "账单问题"
    - name: general
      description: "一般咨询"
  id: classifier

# 技术问题处理
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: system
      text: "你是技术支持专家"
    - role: user
      text: "{{#sys.query#}}"
  id: tech_handler

# 账单问题处理
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: system
      text: "你是账单专家"
    - role: user
      text: "{{#sys.query#}}"
  id: billing_handler

# 一般咨询处理
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: system
      text: "你是客服助手"
    - role: user
      text: "{{#sys.query#}}"
  id: general_handler

# 聚合所有分支的结果
- data:
    type: variable-aggregator
    variables:
    - source: tech_handler
      variable: text
    - source: billing_handler
      variable: text
    - source: general_handler
      variable: text
  id: aggregate_response

# 统一输出（只需一个节点）
- data:
    type: answer
    answer: "{{#aggregate_response.output#}}"
  id: final_answer
```

### 案例2：条件分支聚合

```yaml
# 条件判断
- data:
    type: if-else
    conditions:
    - variable: "{{#start.user_type#}}"
      operator: "equals"
      value: "premium"
  id: check_user_type

# 高级用户处理
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: user
      text: "为高级用户提供详细回答：{{#sys.query#}}"
  id: premium_response

# 普通用户处理
- data:
    type: llm
    model:
      provider: openai
      name: gpt-3.5-turbo
    prompt_template:
    - role: user
      text: "为普通用户提供简洁回答：{{#sys.query#}}"
  id: standard_response

# 聚合响应
- data:
    type: variable-aggregator
    variables:
    - source: premium_response
      variable: text
    - source: standard_response
      variable: text
  id: aggregate_answer

# 后续处理（统一逻辑）
- data:
    type: template
    template: |
      ## 回答
      
      {{#aggregate_answer.output#}}
      
      ---
      来源：{{#aggregate_answer.source#}}
  id: format_output
```

### 案例3：多路径数据处理

```yaml
# 数据源选择
- data:
    type: if-else
    conditions:
    - variable: "{{#start.source#}}"
      operator: "equals"
      value: "database"
  id: select_source

# 从数据库获取
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/db/query"
    body:
      query: "{{#start.query#}}"
  id: fetch_from_db

# 从 API 获取
- data:
    type: http-request
    method: GET
    url: "https://api.example.com/data"
    params:
      q: "{{#start.query#}}"
  id: fetch_from_api

# 从缓存获取
- data:
    type: http-request
    method: GET
    url: "https://cache.example.com/get"
    params:
      key: "{{#start.query#}}"
  id: fetch_from_cache

# 聚合数据
- data:
    type: variable-aggregator
    variables:
    - source: fetch_from_db
      variable: data
    - source: fetch_from_api
      variable: data
    - source: fetch_from_cache
      variable: data
  id: aggregate_data

# 统一处理（无需关心数据来源）
- data:
    type: code
    code: |
      def main(data):
          # 处理数据
          return {"processed": process(data)}
    inputs:
      data: "{{#aggregate_data.output#}}"
  id: process_data
```

### 案例4：多组聚合

```yaml
# 分类处理
- data:
    type: question-classifier
    query: "{{#sys.query#}}"
    classes:
    - name: type_a
    - name: type_b
  id: classifier

# A 类型处理
- data:
    type: llm
    prompt_template:
    - role: user
      text: "处理 A 类型"
  id: handler_a

# B 类型处理
- data:
    type: llm
    prompt_template:
    - role: user
      text: "处理 B 类型"
  id: handler_b

# 多组聚合
- data:
    type: variable-aggregator
    groups:
    - name: responses
      variables:
      - source: handler_a
        variable: text
      - source: handler_b
        variable: text
    - name: metadata
      variables:
      - source: handler_a
        variable: usage
      - source: handler_b
        variable: usage
  id: multi_aggregate

# 使用聚合结果
- data:
    type: template
    template: |
      回答：{{#multi_aggregate.responses#}}
      用量：{{#multi_aggregate.metadata#}}
  id: output
```

## 使用场景

### 分类工作流
- 问题分类处理
- 用户类型路由
- 内容类别处理
- 优先级分流

### 条件分支
- If-Else 结果合并
- 多条件判断
- 异常处理分支
- 降级方案聚合

### 多数据源
- 数据库/API/缓存选择
- 主备切换
- 负载均衡
- 故障转移

### 并行处理
- 多路径执行
- 竞速模式
- 冗余处理
- 结果选择

## 最佳实践

### 1. 类型管理
- 确保所有分支返回相同类型
- 在分支中进行类型转换
- 使用模板节点统一格式
- 验证数据结构

### 2. 分支设计
- 确保只有一个分支执行
- 避免多分支同时激活
- 实现互斥逻辑
- 处理边缘情况

### 3. 命名规范
- 使用清晰的变量名
- 保持分支间命名一致
- 添加注释说明
- 文档化聚合逻辑

### 4. 错误处理
- 处理分支执行失败
- 提供默认值
- 实现降级方案
- 记录错误信息

### 5. 性能优化
- 避免不必要的分支
- 优化分支执行顺序
- 使用缓存减少重复
- 监控执行路径

## 工作流简化

### 不使用聚合器

```yaml
# 需要为每个分支重复下游节点
classifier → branch1 → process1 → output1
          → branch2 → process2 → output2
          → branch3 → process3 → output3
```

### 使用聚合器

```yaml
# 只需一组下游节点
classifier → branch1 ↘
          → branch2 → aggregator → process → output
          → branch3 ↗
```

## 常见问题

### Q: 如何确保类型一致？

A:
1. 在分支中统一输出格式
2. 使用模板节点转换类型
3. 通过代码节点标准化
4. 验证数据结构

### Q: 多个分支同时执行怎么办？

A:
- 聚合器会选择第一个完成的
- 确保分支互斥执行
- 使用条件节点控制
- 检查工作流逻辑

### Q: 如何追踪值的来源？

A:
- 使用 `{{#aggregate.source#}}` 变量
- 在日志中记录来源
- 添加元数据标记
- 实现审计追踪

### Q: 聚合器支持多少个分支？

A:
- 理论上无限制
- 实际取决于工作流复杂度
- 建议不超过 10 个分支
- 考虑性能和可维护性

### Q: 如何处理空值？

A:
1. 在分支中提供默认值
2. 使用 If-Else 检查空值
3. 实现降级逻辑
4. 记录异常情况

## 性能考虑

### 分支数量
- 少量分支（2-3）：最佳
- 中等分支（4-6）：良好
- 大量分支（7+）：考虑重构

### 执行效率
- 只有一个分支执行
- 其他分支被跳过
- 无额外性能开销
- 简化下游处理

## 相关节点

- [Question Classifier 节点](./question-classifier.md) - 问题分类
- [If-Else 节点](./ifelse.md) - 条件分支
- [Variable Assigner 节点](./variable-assigner.md) - 变量赋值

## 返回

[工作流 SKILL](../SKILL.md)

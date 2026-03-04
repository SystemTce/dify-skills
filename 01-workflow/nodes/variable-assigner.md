# 变量赋值器节点详解

## 概述

变量赋值器节点管理会话变量，使其在对话应用的多个对话轮次中持久化，实现有状态的交互。

## 核心功能

- 管理会话级变量
- 跨轮次持久化
- 支持多种操作模式
- 多数据类型支持

## 核心价值

会话变量在整个聊天会话中持久存在，不同于每次执行后重置的工作流变量。这使得能够实现状态记忆、上下文追踪和进度管理。

## 配置元素

### 变量选择
选择要更新的会话变量

### 源数据
从上游工作流节点获取数据

### 操作模式
确定变量如何更新（覆盖、追加、清除等）

## 操作模式

### 字符串操作
- **覆盖**: 替换为新值
- **清除**: 设置为空字符串
- **设置值**: 设置为固定值

```yaml
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.user_name
      operation: overwrite
      value: "{{#start.name#}}"
  id: set_name
```

### 数字操作
- **覆盖**: 替换为新值
- **清除**: 设置为 0
- **设置值**: 设置为固定值
- **加法**: 增加值
- **减法**: 减少值
- **乘法**: 乘以值
- **除法**: 除以值

```yaml
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.score
      operation: add
      value: 10
  id: add_score
```

### 布尔操作
- **覆盖**: 替换为新值
- **清除**: 设置为 false
- **设置值**: 设置为 true/false

```yaml
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.is_verified
      operation: set_value
      value: true
  id: set_verified
```

### 对象操作
- **覆盖**: 替换整个对象
- **清除**: 设置为空对象
- **手动定义**: 自定义对象结构

```yaml
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.user_profile
      operation: overwrite
      value:
        name: "{{#start.name#}}"
        email: "{{#start.email#}}"
        preferences: {}
  id: set_profile
```

### 数组操作
- **覆盖**: 替换整个数组
- **清除**: 设置为空数组
- **追加**: 添加单个元素到末尾
- **扩展**: 合并另一个数组
- **移除首元素**: 删除第一个元素
- **移除尾元素**: 删除最后一个元素

```yaml
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.history
      operation: append
      value: "{{#llm.text#}}"
  id: add_to_history
```

## 实际案例

### 案例1：智能记忆系统

```yaml
# 检测重要信息
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: user
      text: |
        从对话中提取重要信息：
        {{#sys.query#}}
        
        如果有重要信息（姓名、偏好等），以 JSON 格式返回
  id: detect_info

# 保存到会话变量
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.user_facts
      operation: append
      value: "{{#detect_info.text#}}"
  id: save_facts

# 在后续对话中使用
- data:
    type: llm
    prompt_template:
    - role: system
      text: |
        已知用户信息：
        {{#conversation.user_facts#}}
    - role: user
      text: "{{#sys.query#}}"
  id: personalized_response
```

### 案例2：用户偏好存储

```yaml
# 收集偏好
- data:
    type: parameter-extractor
    input: "{{#sys.query#}}"
    parameters:
    - name: language
      type: string
    - name: notifications
      type: boolean
  id: extract_preferences

# 保存偏好
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.preferences
      operation: overwrite
      value:
        language: "{{#extract_preferences.language#}}"
        notifications: "{{#extract_preferences.notifications#}}"
  id: save_preferences
```

### 案例3：进度追踪清单

```yaml
# 初始化任务列表
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.tasks
      operation: overwrite
      value:
      - task: "完成注册"
        status: "pending"
      - task: "验证邮箱"
        status: "pending"
      - task: "设置密码"
        status: "pending"
  id: init_tasks

# 更新任务状态
- data:
    type: code
    code: |
      def main(tasks, completed_task):
          for task in tasks:
              if task['task'] == completed_task:
                  task['status'] = 'completed'
          return {"updated_tasks": tasks}
    inputs:
      tasks: "{{#conversation.tasks#}}"
      completed_task: "{{#start.task_name#}}"
  id: update_task

# 保存更新
- data:
    type: variable-assigner
    assignments:
    - variable: conversation.tasks
      operation: overwrite
      value: "{{#update_task.updated_tasks#}}"
  id: save_tasks

# LLM 引用任务列表
- data:
    type: llm
    prompt_template:
    - role: system
      text: |
        用户任务进度：
        {{#conversation.tasks#}}
        
        引导用户完成剩余任务
  id: guide_user
```

## 使用场景

### 状态管理
- 用户认证状态
- 会话进度追踪
- 临时数据存储
- 上下文保持

### 用户偏好
- 语言设置
- 通知偏好
- 显示选项
- 个性化配置

### 进度追踪
- 多步骤流程
- 任务清单
- 完成状态
- 里程碑记录

### 数据累积
- 对话历史
- 收集的信息
- 统计数据
- 行为记录

## 最佳实践

### 1. 变量命名
- 使用描述性名称
- 遵循命名约定
- 添加前缀分组
- 文档化用途

### 2. 数据结构
- 设计清晰的结构
- 避免过度嵌套
- 保持一致性
- 考虑扩展性

### 3. 操作选择
- 选择合适的操作类型
- 避免不必要的覆盖
- 使用追加而非覆盖
- 及时清理无用数据

### 4. 性能考虑
- 限制变量大小
- 定期清理历史
- 避免存储大对象
- 监控内存使用

## 常见问题

### Q: 会话变量何时清除？

A:
- 会话结束时自动清除
- 可以手动清除操作
- 不会跨会话保留
- 新会话重新初始化

### Q: 如何在工作流中访问会话变量？

A:
使用 `{{#conversation.变量名#}}` 语法引用

### Q: 会话变量有大小限制吗？

A:
- 有内存限制
- 避免存储大量数据
- 定期清理不需要的数据
- 考虑使用外部存储

### Q: 如何调试会话变量？

A:
1. 使用模板节点显示变量
2. 在日志中记录变量值
3. 使用调试模式查看
4. 添加变量监控节点

## 相关节点

- [Variable Aggregator 节点](./variable-aggregator.md) - 变量聚合
- [Parameter Extractor 节点](./parameter-extractor.md) - 参数提取
- [Code 节点](./code.md) - 自定义逻辑

## 返回

[工作流 SKILL](../SKILL.md)

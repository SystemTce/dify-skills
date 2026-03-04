# 模板节点详解

## 概述

模板节点使用 Jinja2 语法将多个来源的数据转换和格式化为结构化文本，用于工作流处理。

## 核心功能

- 变量替换和引用
- 条件渲染
- 循环迭代
- 数据过滤和转换
- 交互式表单生成

## Jinja2 语法

### 变量替换

使用双花括号引用变量：

```jinja2
{{ variable_name }}
{{ user.name }}
{{ items[0].title }}
```

### 嵌套对象访问

```jinja2
{{ api_response.data.items[0].id }}
{{ user.profile.email }}
```

### 数组索引

```jinja2
{{ items[0] }}
{{ items[-1] }}  # 最后一个元素
```

## 条件逻辑

### If-Else 语句

```jinja2
{% if score > 90 %}
优秀
{% elif score > 60 %}
及格
{% else %}
不及格
{% endif %}
```

### 条件渲染

```jinja2
{% if user.is_premium %}
欢迎尊贵的会员！
{% endif %}
```

## 循环迭代

### 基础循环

```jinja2
{% for item in items %}
- {{ item.name }}: {{ item.price }}
{% endfor %}
```

### 带索引的循环

```jinja2
{% for item in items %}
{{ loop.index }}. {{ item.name }}
{% endfor %}
```

### 循环变量

- `loop.index`: 当前迭代（从 1 开始）
- `loop.index0`: 当前迭代（从 0 开始）
- `loop.first`: 是否第一次迭代
- `loop.last`: 是否最后一次迭代
- `loop.length`: 序列长度

## 过滤器

### 文本转换

```jinja2
{{ text | upper }}           # 转大写
{{ text | lower }}           # 转小写
{{ text | title }}           # 标题格式
{{ text | capitalize }}      # 首字母大写
```

### 数字处理

```jinja2
{{ number | round(2) }}      # 四舍五入到2位小数
{{ number | abs }}           # 绝对值
{{ number | int }}           # 转整数
{{ number | float }}         # 转浮点数
```

### 字符串操作

```jinja2
{{ text | replace("old", "new") }}  # 替换
{{ text | trim }}                   # 去除首尾空格
{{ text | truncate(100) }}          # 截断到100字符
```

### 日期格式化

```jinja2
{{ timestamp | strftime("%Y-%m-%d") }}
{{ timestamp | strftime("%H:%M:%S") }}
```

### 默认值

```jinja2
{{ variable | default("默认值") }}
```

## 交互式表单

模板可以生成 HTML 表单，在聊天界面中收集结构化数据：

```jinja2
<form>
  <label>姓名：</label>
  <input type="text" name="name" required />

  <label>年龄：</label>
  <input type="number" name="age" min="18" max="100" />

  <label>性别：</label>
  <select name="gender">
    <option value="male">男</option>
    <option value="female">女</option>
  </select>

  <button type="submit">提交</button>
</form>
```

用户提交后，响应变为 JSON 数据供下游处理。

## 技术约束

输出限制为 80,000 字符（可通过 `TEMPLATE_TRANSFORM_MAX_LENGTH` 配置）。

## 实际案例

### 案例1：报告生成

```jinja2
# {{ report_title }}

## 执行摘要

生成时间：{{ timestamp | strftime("%Y-%m-%d %H:%M:%S") }}
分析项目：{{ project_name }}

## 数据统计

- 总记录数：{{ total_count }}
- 成功数：{{ success_count }}
- 失败数：{{ failed_count }}
- 成功率：{{ (success_count / total_count * 100) | round(2) }}%

## 详细结果

{% for item in results %}
### {{ loop.index }}. {{ item.name }}

- 状态：{{ item.status | upper }}
- 耗时：{{ item.duration }}秒
- 结果：{{ item.result }}

{% endfor %}

## 建议

{% if success_count / total_count < 0.8 %}
⚠️ 成功率低于80%，建议检查以下方面：
- 数据质量
- 系统配置
- 网络连接
{% else %}
✓ 系统运行正常
{% endif %}
```

### 案例2：API 响应格式化

```jinja2
{
  "status": "success",
  "data": {
    "user_id": "{{ user.id }}",
    "username": "{{ user.name }}",
    "email": "{{ user.email }}",
    "profile": {
      "age": {{ user.age }},
      "location": "{{ user.location }}",
      "premium": {{ user.is_premium | lower }}
    },
    "statistics": {
      "total_orders": {{ orders | length }},
      "total_spent": {{ total_spent | round(2) }}
    }
  },
  "timestamp": "{{ now | strftime("%Y-%m-%dT%H:%M:%SZ") }}"
}
```

### 案例3：LLM Prompt 准备

```jinja2
你是一个专业的{{ role }}。

用户信息：
- 姓名：{{ user.name }}
- 等级：{{ user.level }}
- 偏好：{{ user.preferences | join(", ") }}

历史对话：
{% for msg in conversation_history[-5:] %}
{{ msg.role }}: {{ msg.content }}
{% endfor %}

当前任务：
{{ task_description }}

要求：
{% for requirement in requirements %}
- {{ requirement }}
{% endfor %}

请基于以上信息完成任务。
```

### 案例4：个性化邮件模板

```jinja2
尊敬的 {{ customer.name }}，

感谢您在 {{ order.date | strftime("%Y年%m月%d日") }} 的订单！

订单详情：
订单号：{{ order.id }}

{% for item in order.items %}
- {{ item.name }} x {{ item.quantity }} = ¥{{ (item.price * item.quantity) | round(2) }}
{% endfor %}

总计：¥{{ order.total | round(2) }}

{% if order.total > 500 %}
🎉 恭喜您！本次订单享受免运费优惠。
{% endif %}

预计送达时间：{{ delivery_date | strftime("%Y年%m月%d日") }}

如有任何问题，请随时联系我们。

祝您购物愉快！

{{ company_name }}
{{ company_email }}
```

### 案例5：数据转换

```jinja2
{% set processed_items = [] %}
{% for item in raw_data %}
  {% if item.status == "active" and item.score > 60 %}
    {% set _ = processed_items.append({
      "id": item.id,
      "name": item.name | title,
      "score": item.score | round(2),
      "grade": "优秀" if item.score > 90 else ("良好" if item.score > 75 else "及格")
    }) %}
  {% endif %}
{% endfor %}

处理结果：
共处理 {{ raw_data | length }} 条记录
筛选出 {{ processed_items | length }} 条符合条件的记录

{% for item in processed_items %}
{{ loop.index }}. {{ item.name }} - {{ item.grade }} ({{ item.score }}分)
{% endfor %}
```

### 案例6：交互式表单

```jinja2
<form>
  <h2>用户反馈表单</h2>

  <label>您的姓名：</label>
  <input type="text" name="name" required />

  <label>联系邮箱：</label>
  <input type="email" name="email" required />

  <label>反馈类型：</label>
  <select name="feedback_type" required>
    <option value="">请选择</option>
    <option value="bug">Bug 报告</option>
    <option value="feature">功能建议</option>
    <option value="question">问题咨询</option>
    <option value="other">其他</option>
  </select>

  <label>详细描述：</label>
  <textarea name="description" rows="5" required></textarea>

  <label>紧急程度：</label>
  <input type="range" name="urgency" min="1" max="5" value="3" />

  <button type="submit">提交反馈</button>
</form>
```

## 实际应用

### 报告生成

组合多个数据源生成综合报告：

```jinja2
# 数据分析报告

## 概览
- 数据来源：{{ source }}
- 分析时间：{{ timestamp }}
- 记录总数：{{ total }}

## 统计
{% for stat in statistics %}
- {{ stat.name }}: {{ stat.value }}
{% endfor %}
```

### API 响应格式化

将内部数据转换为外部 API 格式：

```jinja2
{
  "status": "{{ status }}",
  "data": {{ data | tojson }},
  "meta": {
    "timestamp": "{{ now }}",
    "version": "1.0"
  }
}
```

### LLM Prompt 准备

构建复杂的 LLM 提示词：

```jinja2
角色：{{ role }}
任务：{{ task }}
上下文：
{% for ctx in context %}
- {{ ctx }}
{% endfor %}
```

### 个性化内容

生成个性化的邮件或消息：

```jinja2
Hi {{ user.name }},

{% if user.is_premium %}
作为我们的尊贵会员...
{% else %}
感谢您的支持...
{% endif %}
```

### 数据转换

为外部系统集成转换数据格式：

```jinja2
{% for item in items %}
{{ item.id }},{{ item.name }},{{ item.value }}
{% endfor %}
```

## 最佳实践

### 1. 变量命名
- 使用描述性的变量名
- 保持命名一致性
- 避免使用保留关键字

### 2. 错误处理
- 使用默认值处理缺失数据
- 添加条件检查
- 验证数据类型

```jinja2
{{ variable | default("N/A") }}

{% if variable %}
  {{ variable }}
{% else %}
  数据不可用
{% endif %}
```

### 3. 格式化输出
- 使用适当的过滤器
- 保持输出可读性
- 考虑目标格式要求

### 4. 性能考虑
- 避免过度嵌套循环
- 限制输出长度
- 使用简洁的表达式

### 5. 测试
- 使用示例数据测试模板
- 处理边界情况
- 验证输出格式

## 常见问题

### Q: 如何处理缺失的变量？

A: 使用 `default` 过滤器：
```jinja2
{{ variable | default("默认值") }}
```

### Q: 如何格式化日期？

A: 使用 `strftime` 过滤器：
```jinja2
{{ timestamp | strftime("%Y-%m-%d %H:%M:%S") }}
```

### Q: 如何处理 JSON 数据？

A: 使用 `tojson` 过滤器：
```jinja2
{{ data | tojson }}
```

### Q: 模板输出太长怎么办？

A:
1. 使用 `truncate` 过滤器截断
2. 限制循环次数
3. 调整 `TEMPLATE_TRANSFORM_MAX_LENGTH` 配置

### Q: 如何调试模板？

A:
1. 使用简单的测试数据
2. 逐步添加复杂逻辑
3. 检查变量是否存在
4. 验证过滤器语法

## 相关节点

- [Code 节点](./code.md) - 复杂数据处理
- [LLM 节点](./llm.md) - 使用模板准备 Prompt
- [HTTP 请求节点](./http-request.md) - 格式化 API 请求

## 返回

[工作流 SKILL](../SKILL.md)

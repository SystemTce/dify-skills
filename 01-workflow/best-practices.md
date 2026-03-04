# Dify 工作流最佳实践

## 概述

本文档总结了 Dify 工作流设计和开发的最佳实践，帮助开发者构建高质量、高性能、易维护的工作流应用。

## 一、设计原则

### 1. 模块化设计

**原则**: 将复杂流程拆分为独立的、可复用的模块。

**实践**:
- 每个节点专注于单一职责
- 使用变量传递数据，避免硬编码
- 创建可复用的子工作流

**示例**:
```yaml
# 不好：单个节点做太多事情
Code节点: 数据清洗 + 验证 + 转换 + 分析 + 格式化

# 好：拆分为多个节点
Code1: 数据清洗
Code2: 数据验证
LLM: 数据分析
Code3: 结果格式化
```

### 2. 渐进式复杂度

**原则**: 从简单开始，逐步增加复杂度。

**实践**:
- 先实现核心功能
- 验证基础流程
- 再添加高级特性

**流程**:
1. 简单管道：Start → LLM → Answer
2. 添加验证：Start → Code → LLM → Answer
3. 添加分支：Start → Code → If-Else → [LLM1 / LLM2] → Answer
4. 添加迭代：Start → Code → Iteration → LLM → Answer

### 3. 错误优先设计

**原则**: 提前考虑错误情况和边界条件。

**实践**:
- 验证输入数据
- 处理空值和异常
- 提供降级方案
- 记录错误信息

**示例**:
```python
def main(data: str) -> dict:
    # 输入验证
    if not data:
        return {"error": "输入为空", "success": False}

    if len(data) > 10000:
        return {"error": "输入过长", "success": False}

    try:
        # 处理逻辑
        result = process(data)
        return {"result": result, "success": True}
    except Exception as e:
        return {"error": str(e), "success": False}
```

## 二、节点使用

### 1. Start 节点

**最佳实践**:
- 使用清晰的变量名
- 提供输入说明
- 设置合理的长度限制
- 标记必填字段

**示例**:
```yaml
variables:
- label: 用户问题
  variable: user_query
  type: text-input
  required: true
  max_length: 500
  description: 请输入您的问题（不超过500字）
```

### 2. LLM 节点

**最佳实践**:
- 明确定义角色和任务
- 使用结构化提示词
- 设置合理的 temperature
- 控制 max_tokens

**提示词模板**:
```yaml
prompt_template:
- role: system
  text: |
    你是[角色定义]。

    你的职责：
    1. [职责1]
    2. [职责2]

    你的风格：
    - [风格1]
    - [风格2]

- role: user
  text: |
    [上下文信息]
    {{#context#}}

    [用户输入]
    {{#user_input#}}

    [输出要求]
    请按照以下格式输出：
    [格式说明]
```

### 3. Code 节点

**最佳实践**:
- 添加类型注解
- 编写文档字符串
- 处理异常情况
- 返回结构化数据

**代码模板**:
```python
def main(param1: str, param2: int) -> dict:
    """
    函数说明

    Args:
        param1: 参数1说明
        param2: 参数2说明

    Returns:
        dict: {
            "success": bool,
            "data": any,
            "error": str (optional)
        }
    """
    try:
        # 输入验证
        if not param1:
            return {"success": False, "error": "param1不能为空"}

        # 处理逻辑
        result = process_data(param1, param2)

        return {
            "success": True,
            "data": result
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"处理失败: {str(e)}"
        }
```

### 4. If-Else 节点

**最佳实践**:
- 保持条件简单
- 处理所有分支
- 使用有意义的分支名
- 避免过深嵌套

**示例**:
```yaml
# 简单条件
conditions:
- variable_selector: ["code", "success"]
  comparison_operator: is
  value: true

# 复杂逻辑用 Code 节点
def main(data: dict) -> dict:
    # 复杂判断逻辑
    is_valid = (
        data.get("score", 0) > 80 and
        data.get("status") == "active" and
        len(data.get("items", [])) > 0
    )
    return {"is_valid": is_valid}
```

### 5. Iteration 节点

**最佳实践**:
- 控制批量大小
- 设置合理的并行数
- 使用 continue 错误模式
- 聚合处理结果

**示例**:
```yaml
# 迭代配置
parallel_nums: 3  # 根据任务类型调整
error_handle_mode: continue  # 避免单个失败影响整体

# 结果聚合
def main(results: list) -> dict:
    success = [r for r in results if r.get("success")]
    failed = [r for r in results if not r.get("success")]

    return {
        "total": len(results),
        "success_count": len(success),
        "failed_count": len(failed),
        "success_rate": len(success) / len(results) * 100,
        "data": success
    }
```

## 三、变量管理

### 1. 命名规范

**原则**: 使用有意义的、一致的变量名。

**规范**:
- 使用 snake_case（下划线分隔）
- 避免缩写（除非是通用缩写）
- 使用描述性名称

**示例**:
```yaml
# 不好
var1, x, temp, data

# 好
user_query, analysis_result, cleaned_text, api_response
```

### 2. 变量作用域

**理解作用域**:
- 系统变量：全局可用
- 节点变量：当前节点及后续节点
- 迭代变量：仅在迭代内部
- 会话变量：跨轮对话

**示例**:
```yaml
{{#sys.query#}}              # 系统变量
{{#start.user_input#}}       # Start 节点变量
{{#llm.text#}}               # LLM 节点变量
{{#item#}}                   # 迭代变量
{{#conversation.user_name#}} # 会话变量
```

### 3. 变量传递

**最佳实践**:
- 明确数据流向
- 避免循环依赖
- 使用中间变量简化引用

**示例**:
```python
# 使用中间变量
def main(llm_output: str, code_output: dict) -> dict:
    # 整合数据
    combined = {
        "analysis": llm_output,
        "stats": code_output,
        "timestamp": datetime.now().isoformat()
    }
    return {"result": combined}
```

## 四、性能优化

### 1. 减少 LLM 调用

**策略**:
- 合并多个 LLM 任务
- 使用缓存
- 选择合适的模型

**示例**:
```yaml
# 不好：多次 LLM 调用
LLM1: 提取关键词
LLM2: 分析情感
LLM3: 生成摘要

# 好：单次 LLM 调用
LLM: 提取关键词 + 分析情感 + 生成摘要
```

### 2. 并行执行

**策略**:
- 识别独立任务
- 使用并行分支
- 合理设置并行数

**示例**:
```yaml
# 并行执行独立任务
Start → [HTTP1, HTTP2, HTTP3] → Variable Aggregator → LLM → Answer
```

### 3. 数据处理优化

**策略**:
- 使用 Code 节点预处理
- 减少数据传输量
- 批量处理

**示例**:
```python
# 预处理减少 LLM 输入
def main(raw_text: str) -> dict:
    # 清洗和压缩
    cleaned = raw_text.strip()
    # 提取关键部分
    key_parts = extract_key_sections(cleaned)
    # 只传递必要信息给 LLM
    return {"processed_text": key_parts}
```

### 4. 缓存策略

**策略**:
- 对相同输入启用缓存
- 缓存知识库检索结果
- 使用会话变量存储中间结果

## 五、错误处理

### 1. 输入验证

**最佳实践**:
```python
def main(input_data: str) -> dict:
    # 空值检查
    if not input_data:
        return {"error": "输入不能为空"}

    # 长度检查
    if len(input_data) > 10000:
        return {"error": "输入过长"}

    # 格式检查
    if not is_valid_format(input_data):
        return {"error": "格式不正确"}

    # 处理逻辑
    return {"result": process(input_data)}
```

### 2. 异常捕获

**最佳实践**:
```python
def main(data: dict) -> dict:
    try:
        result = risky_operation(data)
        return {"success": True, "data": result}
    except ValueError as e:
        return {"success": False, "error": f"数据错误: {str(e)}"}
    except KeyError as e:
        return {"success": False, "error": f"缺少字段: {str(e)}"}
    except Exception as e:
        return {"success": False, "error": f"未知错误: {str(e)}"}
```

### 3. 降级策略

**最佳实践**:
```yaml
# 使用 If-Else 实现降级
If-Else:
  条件: 主要服务成功
  True: 使用主要结果
  False: 使用备用方案
```

### 4. 错误提示

**最佳实践**:
- 提供清晰的错误信息
- 说明错误原因
- 给出解决建议

**示例**:
```python
return {
    "error": "API 调用失败",
    "reason": "连接超时",
    "suggestion": "请检查网络连接或稍后重试"
}
```

## 六、可维护性

### 1. 文档和注释

**最佳实践**:
- 为每个节点添加描述
- 在 Code 节点中添加注释
- 记录设计决策

**示例**:
```yaml
# 节点描述
- data:
    desc: '清洗用户输入，移除特殊字符和多余空格'
    type: code
```

```python
def main(text: str) -> dict:
    """
    清洗文本数据

    处理步骤：
    1. 移除首尾空格
    2. 转换为小写
    3. 移除特殊字符
    4. 合并多余空格

    Args:
        text: 原始文本

    Returns:
        dict: {"cleaned_text": str}
    """
    # 实现逻辑
```

### 2. 版本控制

**最佳实践**:
- 使用 Git 管理 DSL 文件
- 记录版本变更
- 保留重要版本

**示例**:
```bash
# 导出工作流
dify export workflow --id xxx --output workflow-v1.0.yml

# 提交到 Git
git add workflow-v1.0.yml
git commit -m "feat: 添加错误处理逻辑"
git tag v1.0
```

### 3. 测试策略

**最佳实践**:
- 测试正常流程
- 测试边界条件
- 测试错误情况

**测试清单**:
- [ ] 正常输入测试
- [ ] 空输入测试
- [ ] 超长输入测试
- [ ] 特殊字符测试
- [ ] 并发测试
- [ ] 性能测试

### 4. 监控和日志

**最佳实践**:
- 记录关键节点的执行
- 监控 Token 使用
- 追踪错误率

**示例**:
```python
def main(data: dict) -> dict:
    # 记录输入
    print(f"Input: {data}")

    result = process(data)

    # 记录输出
    print(f"Output: {result}")

    return result
```

## 七、安全实践

### 1. 输入过滤

**最佳实践**:
- 过滤敏感信息
- 验证输入格式
- 限制输入长度

### 2. 环境变量

**最佳实践**:
- 使用环境变量存储密钥
- 不在代码中硬编码敏感信息

**示例**:
```yaml
environment_variables:
- name: API_KEY
  value: "{{#env.API_KEY#}}"

# 在节点中使用
headers:
  Authorization: "Bearer {{#env.API_KEY#}}"
```

### 3. 权限控制

**最佳实践**:
- 限制 API 访问权限
- 使用最小权限原则
- 定期轮换密钥

## 八、成本优化

### 1. 模型选择

**策略**:
- 简单任务：GPT-3.5-turbo、Qwen-Turbo
- 复杂任务：GPT-4、Claude 3 Opus
- 代码任务：DeepSeek-Coder

### 2. Token 优化

**策略**:
- 精简提示词
- 设置 max_tokens
- 使用缓存

### 3. 批量处理

**策略**:
- 合并相似请求
- 使用迭代节点
- 控制并行数

## 九、团队协作

### 1. 命名规范

**统一规范**:
- 工作流命名：`项目-功能-版本`
- 节点命名：`动词_名词`
- 变量命名：`snake_case`

### 2. 代码审查

**检查清单**:
- [ ] 是否遵循命名规范
- [ ] 是否有充分的错误处理
- [ ] 是否有必要的注释
- [ ] 是否有性能问题
- [ ] 是否有安全隐患

### 3. 知识分享

**实践**:
- 文档化最佳实践
- 分享成功案例
- 定期技术分享

## 十、持续改进

### 1. 性能监控

**指标**:
- 执行时间
- Token 消耗
- 错误率
- 用户满意度

### 2. 用户反馈

**收集方式**:
- 用户评分
- 错误报告
- 功能建议

### 3. 迭代优化

**流程**:
1. 收集数据和反馈
2. 识别改进点
3. 实施优化
4. 验证效果
5. 持续监控

## 总结

遵循这些最佳实践，可以帮助你构建高质量、高性能、易维护的 Dify 工作流应用。记住：

- **简单优先**: 从简单开始，逐步增加复杂度
- **错误优先**: 提前考虑错误情况
- **性能优先**: 关注性能和成本
- **可维护优先**: 编写清晰、有文档的代码
- **持续改进**: 基于数据和反馈不断优化

## 返回

[工作流 SKILL](./SKILL.md)

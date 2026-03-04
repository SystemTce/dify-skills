# Code 节点详解

## 概述

Code 节点允许在工作流中执行 Python 或 JavaScript 代码，用于数据处理、格式转换、逻辑计算等任务。

## 核心功能

- 执行 Python 3 或 JavaScript 代码
- 数据转换和处理
- 复杂逻辑计算
- API 数据解析
- 自定义业务逻辑

## 配置结构

### Python 代码节点

```yaml
- data:
    type: code
    title: 数据处理
    code_language: python3
    code: |
      def main(input_text: str, count: int) -> dict:
          # 处理逻辑
          result = input_text.upper()
          items = [f"Item {i}" for i in range(count)]

          return {
              "output": result,
              "items": items,
              "length": len(result)
          }
    variables:
    - variable: input_text
      value_selector: ["start", "user_input"]
    - variable: count
      value_selector: ["start", "count"]
    outputs:
      output:
        type: string
      items:
        type: array[string]
      length:
        type: number
  id: code
```

### JavaScript 代码节点

```yaml
- data:
    type: code
    code_language: javascript
    code: |
      function main(data) {
          const result = data.text.toUpperCase();
          return {
              output: result,
              timestamp: Date.now()
          };
      }
    variables:
    - variable: data
      value_selector: ["start", "input"]
    outputs:
      output:
        type: string
      timestamp:
        type: number
  id: js_code
```

## 输入变量

### 变量配置

```yaml
variables:
- variable: text        # 参数名
  value_selector: ["start", "user_input"]  # 数据来源
- variable: options
  value_selector: ["llm", "text"]
```

### 支持的数据类型

- `string`: 字符串
- `number`: 数字
- `boolean`: 布尔值
- `object`: 对象/字典
- `array`: 数组/列表
- `file`: 文件对象

## 输出定义

### 输出类型

```yaml
outputs:
  result:
    type: string
    description: 处理结果
  count:
    type: number
    description: 数量统计
  items:
    type: array[object]
    description: 数据列表
  success:
    type: boolean
    description: 是否成功
```

## Python 代码示例

### 示例1：文本处理

```python
def main(text: str) -> dict:
    """文本清洗和分析"""
    # 清洗文本
    cleaned = text.strip().lower()

    # 统计信息
    word_count = len(cleaned.split())
    char_count = len(cleaned)

    # 提取关键词（简单示例）
    words = cleaned.split()
    keywords = [w for w in words if len(w) > 5]

    return {
        "cleaned_text": cleaned,
        "word_count": word_count,
        "char_count": char_count,
        "keywords": keywords[:5]
    }
```

### 示例2：JSON 数据解析

```python
import json

def main(json_str: str) -> dict:
    """解析和转换 JSON 数据"""
    try:
        data = json.loads(json_str)

        # 提取需要的字段
        result = {
            "id": data.get("id"),
            "name": data.get("name"),
            "items": [item["title"] for item in data.get("items", [])]
        }

        return {
            "success": True,
            "data": result
        }
    except Exception as e:
        return {
            "success": False,
            "error": str(e)
        }
```

### 示例3：数组处理

```python
def main(items: list) -> dict:
    """批量数据处理"""
    # 过滤和转换
    filtered = [item for item in items if item.get("status") == "active"]
    transformed = [
        {
            "id": item["id"],
            "name": item["name"].upper(),
            "score": item.get("score", 0) * 1.1
        }
        for item in filtered
    ]

    # 统计
    total_score = sum(item["score"] for item in transformed)
    avg_score = total_score / len(transformed) if transformed else 0

    return {
        "items": transformed,
        "count": len(transformed),
        "total_score": total_score,
        "avg_score": avg_score
    }
```

### 示例4：日期时间处理

```python
from datetime import datetime, timedelta

def main(date_str: str) -> dict:
    """日期时间计算"""
    # 解析日期
    date = datetime.strptime(date_str, "%Y-%m-%d")

    # 计算
    today = datetime.now()
    days_diff = (today - date).days
    next_week = date + timedelta(days=7)

    return {
        "original": date_str,
        "days_ago": days_diff,
        "next_week": next_week.strftime("%Y-%m-%d"),
        "is_past": days_diff > 0
    }
```

### 示例5：数学计算

```python
import math

def main(numbers: list) -> dict:
    """统计分析"""
    if not numbers:
        return {"error": "Empty list"}

    # 基础统计
    total = sum(numbers)
    count = len(numbers)
    mean = total / count

    # 标准差
    variance = sum((x - mean) ** 2 for x in numbers) / count
    std_dev = math.sqrt(variance)

    # 最值
    min_val = min(numbers)
    max_val = max(numbers)

    return {
        "count": count,
        "sum": total,
        "mean": mean,
        "std_dev": std_dev,
        "min": min_val,
        "max": max_val,
        "range": max_val - min_val
    }
```

## JavaScript 代码示例

### 示例1：数据转换

```javascript
function main(data) {
    // 转换数据格式
    const transformed = data.items.map(item => ({
        id: item.id,
        title: item.name.toUpperCase(),
        value: parseFloat(item.price) * 1.1
    }));

    return {
        items: transformed,
        count: transformed.length
    };
}
```

### 示例2：字符串操作

```javascript
function main(text) {
    // 文本处理
    const words = text.split(' ');
    const capitalized = words.map(w =>
        w.charAt(0).toUpperCase() + w.slice(1).toLowerCase()
    ).join(' ');

    return {
        original: text,
        capitalized: capitalized,
        word_count: words.length
    };
}
```

## 可用的 Python 库

### 内置库

- `json`: JSON 处理
- `re`: 正则表达式
- `math`: 数学函数
- `datetime`: 日期时间
- `random`: 随机数
- `base64`: Base64 编码
- `hashlib`: 哈希函数
- `urllib`: URL 处理

### 第三方库

- `jieba`: 中文分词
- `json_repair`: JSON 修复

## 最佳实践

### 1. 函数签名

```python
def main(param1: str, param2: int) -> dict:
    """
    函数说明

    Args:
        param1: 参数1说明
        param2: 参数2说明

    Returns:
        dict: 返回值说明
    """
    pass
```

### 2. 错误处理

```python
def main(data: str) -> dict:
    try:
        result = process_data(data)
        return {
            "success": True,
            "data": result
        }
    except ValueError as e:
        return {
            "success": False,
            "error": f"数据格式错误: {str(e)}"
        }
    except Exception as e:
        return {
            "success": False,
            "error": f"处理失败: {str(e)}"
        }
```

### 3. 数据验证

```python
def main(items: list) -> dict:
    # 验证输入
    if not items:
        return {"error": "输入列表为空"}

    if not isinstance(items, list):
        return {"error": "输入必须是列表"}

    # 处理数据
    result = process_items(items)

    return {"data": result}
```

### 4. 性能优化

```python
def main(data: list) -> dict:
    # 使用列表推导式（更快）
    result = [item * 2 for item in data if item > 0]

    # 避免重复计算
    length = len(result)
    total = sum(result)

    return {
        "result": result,
        "count": length,
        "average": total / length if length > 0 else 0
    }
```

### 5. 代码组织

```python
def main(data: dict) -> dict:
    # 辅助函数
    def validate(item):
        return item.get("status") == "active"

    def transform(item):
        return {
            "id": item["id"],
            "name": item["name"].upper()
        }

    # 主逻辑
    valid_items = [item for item in data["items"] if validate(item)]
    transformed = [transform(item) for item in valid_items]

    return {"items": transformed}
```

## 限制和注意事项

### 1. 执行限制

- 执行时间限制：30 秒
- 内存限制：根据部署配置
- 不支持文件系统操作
- 不支持网络请求（使用 HTTP Request 节点）

### 2. 安全限制

- 代码在沙箱环境中执行
- 无法访问系统资源
- 无法导入未授权的库

### 3. 数据类型

- 输入和输出必须是可序列化的
- 避免使用复杂的自定义对象
- 大数据量建议分批处理

## 常见问题

### Q: 如何处理大数据量？

A: 使用迭代节点分批处理，每批数据在 Code 节点中处理。

### Q: 可以调用外部 API 吗？

A: 不可以，使用 HTTP Request 节点调用外部 API。

### Q: 如何调试代码？

A: 使用 `print()` 输出调试信息，或在返回值中包含调试数据。

### Q: 支持异步代码吗？

A: Python 支持 `async/await`，但建议使用同步代码以简化逻辑。

## 相关节点

- [HTTP Request 节点](./http-request.md) - 调用外部 API
- [Iteration 节点](./iteration.md) - 批量处理数据
- [LLM 节点](./llm.md) - 数据预处理和后处理

## 返回

[工作流 SKILL](../SKILL.md)

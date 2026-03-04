# 图结构优化详解

## 概述

工作流的图结构直接影响执行效率。本文档详细介绍如何通过优化图结构提升工作流性能。

## 核心原则

1. **最小化串行依赖**: 减少必须顺序执行的节点
2. **最大化并行执行**: 识别可以并行的节点
3. **简化图结构**: 移除不必要的节点和连接
4. **合理分组**: 将相关节点组织在一起

## 图结构分析

### 识别串行依赖

**工具**: 依赖关系分析

```python
def analyze_dependencies(workflow):
    """分析工作流的依赖关系"""
    dependencies = {}
    for node in workflow.nodes:
        dependencies[node.id] = {
            "depends_on": node.inputs,
            "depended_by": []
        }

    # 构建反向依赖
    for node_id, deps in dependencies.items():
        for dep in deps["depends_on"]:
            dependencies[dep]["depended_by"].append(node_id)

    return dependencies
```

### 计算关键路径

关键路径是工作流中最长的执行路径,决定了最小执行时间。

```python
def calculate_critical_path(workflow):
    """计算关键路径"""
    # 拓扑排序
    sorted_nodes = topological_sort(workflow)

    # 计算每个节点的最早开始时间
    earliest_start = {}
    for node in sorted_nodes:
        if not node.dependencies:
            earliest_start[node.id] = 0
        else:
            earliest_start[node.id] = max(
                earliest_start[dep] + get_node_duration(dep)
                for dep in node.dependencies
            )

    # 找出关键路径
    critical_path = []
    current = max(earliest_start, key=earliest_start.get)
    while current:
        critical_path.append(current)
        current = get_predecessor_on_critical_path(current)

    return critical_path
```

## 并行化策略

### 策略 1: 数据并行

将数据分片,并行处理每个分片。

**示例**: 批量文档处理

```yaml
# 原始流程 (串行)
Start → 迭代文档 → 处理文档 → 汇总结果 → End

# 优化流程 (并行)
Start → 分片文档 → [并行处理分片1, 并行处理分片2, ...] → 汇总结果 → End
```

### 策略 2: 任务并行

将独立的任务并行执行。

**示例**: 多源数据获取

```yaml
# 原始流程 (串行)
Start → API调用1 → API调用2 → API调用3 → 数据聚合 → End

# 优化流程 (并行)
Start → [API调用1, API调用2, API调用3] → 数据聚合 → End
```

### 策略 3: 流水线并行

将处理流程分段,每段并行处理不同数据。

**示例**: 数据处理流水线

```yaml
# 流水线结构
阶段1: 数据获取 → 队列1
阶段2: 队列1 → 数据清洗 → 队列2
阶段3: 队列2 → 数据分析 → 队列3
阶段4: 队列3 → 结果输出
```

## 图结构模式

### 模式 1: 扇出-扇入 (Fan-out Fan-in)

**适用场景**: 需要并行处理多个独立任务,然后聚合结果

```yaml
        ┌─→ 任务1 ─┐
Start ──┼─→ 任务2 ─┼─→ 聚合 → End
        └─→ 任务3 ─┘
```

**实现**:

```yaml
nodes:
  - id: start
    type: start

  - id: task1
    type: http-request
    depends_on: [start]

  - id: task2
    type: http-request
    depends_on: [start]

  - id: task3
    type: http-request
    depends_on: [start]

  - id: aggregate
    type: variable-aggregator
    depends_on: [task1, task2, task3]

  - id: end
    type: answer
    depends_on: [aggregate]
```

### 模式 2: 条件分支

**适用场景**: 根据条件选择不同的处理路径

```yaml
        ┌─→ 路径A ─┐
Start ──┤          ├─→ End
        └─→ 路径B ─┘
```

**实现**:

```yaml
nodes:
  - id: start
    type: start

  - id: condition
    type: if-else
    depends_on: [start]
    config:
      condition: "{{#start.output.type#}} == 'A'"

  - id: path_a
    type: llm
    depends_on: [condition]
    condition: true

  - id: path_b
    type: llm
    depends_on: [condition]
    condition: false

  - id: end
    type: answer
    depends_on: [path_a, path_b]
```

### 模式 3: 迭代处理

**适用场景**: 需要对数组数据进行批量处理

```yaml
Start → 迭代器 → [处理项1, 处理项2, ...] → 汇总 → End
```

**实现**:

```yaml
nodes:
  - id: start
    type: start

  - id: iterator
    type: iteration
    depends_on: [start]
    config:
      items: "{{#start.output.items#}}"
      parallel_nums: 5  # 并行处理5个项目

  - id: process_item
    type: llm
    depends_on: [iterator]
    inputs:
      item: "{{#iterator.item#}}"

  - id: aggregate
    type: code
    depends_on: [iterator]

  - id: end
    type: answer
    depends_on: [aggregate]
```

## 优化技巧

### 技巧 1: 合并相似节点

**问题**: 多个相似的节点增加图复杂度

```yaml
# 优化前
Start → 转换1 → 转换2 → 转换3 → End

# 优化后
Start → 统一转换 → End
```

### 技巧 2: 提前过滤

**问题**: 在处理后期才过滤数据,浪费资源

```yaml
# 优化前
Start → 获取数据 → 处理数据 → 过滤数据 → End

# 优化后
Start → 获取数据 → 过滤数据 → 处理数据 → End
```

### 技巧 3: 缓存中间结果

**问题**: 重复计算相同的中间结果

```yaml
# 优化前
Start → 计算A → 使用A → 计算A → 使用A → End

# 优化后
Start → 计算A → [使用A, 使用A] → End
```

## 性能基准

### 测试场景

**场景 1**: 10 个独立 API 调用

| 结构 | 执行时间 | 提升 |
|-----|---------|-----|
| 串行 | 20s | - |
| 并行 (5) | 4s | 5x |
| 并行 (10) | 2s | 10x |

**场景 2**: 100 个文档处理

| 结构 | 执行时间 | 提升 |
|-----|---------|-----|
| 串行 | 200s | - |
| 并行 (5) | 40s | 5x |
| 并行 (10) | 20s | 10x |

## 最佳实践

1. **先分析再优化**: 使用依赖分析工具识别瓶颈
2. **渐进式优化**: 从影响最大的部分开始
3. **保持简洁**: 避免过度复杂的图结构
4. **文档化**: 记录优化决策和效果
5. **持续监控**: 跟踪优化后的性能指标

## 工具和脚本

### 图结构可视化

```python
import networkx as nx
import matplotlib.pyplot as plt

def visualize_workflow(workflow):
    """可视化工作流图结构"""
    G = nx.DiGraph()

    # 添加节点
    for node in workflow.nodes:
        G.add_node(node.id, label=node.type)

    # 添加边
    for node in workflow.nodes:
        for dep in node.dependencies:
            G.add_edge(dep, node.id)

    # 绘制图
    pos = nx.spring_layout(G)
    nx.draw(G, pos, with_labels=True, node_color='lightblue',
            node_size=1000, font_size=10, arrows=True)
    plt.show()
```

### 性能分析

```python
def analyze_performance(workflow, execution_log):
    """分析工作流性能"""
    # 计算每个节点的执行时间
    node_times = {}
    for entry in execution_log:
        node_times[entry.node_id] = entry.duration

    # 识别瓶颈节点
    bottlenecks = sorted(node_times.items(),
                        key=lambda x: x[1],
                        reverse=True)[:5]

    # 计算并行度
    parallelism = calculate_parallelism(workflow)

    return {
        "bottlenecks": bottlenecks,
        "parallelism": parallelism,
        "total_time": sum(node_times.values()),
        "critical_path_time": calculate_critical_path_time(workflow)
    }
```

## 参考资料

- [Dify 并行分支功能](https://dify.ai/blog/accelerating-workflow-processing-with-parallel-branch)
- [工作流设计模式](../../01-workflow/patterns/)
- [性能监控指南](../monitoring.md)

---

**最后更新**: 2026-03-04

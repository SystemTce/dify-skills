# Dify SKILL 工程

> 一个模块化、渐进式披露的 Dify 工作流和插件开发技能体系

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/your-repo/dify-skills)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Dify](https://img.shields.io/badge/Dify-0.15.x--1.9.x-orange.svg)](https://github.com/langgenius/dify)

## 简介

Dify SKILL 工程是一个为 Dify 开发者设计的完整技能体系，通过模块化、渐进式披露的方式，帮助开发者快速掌握 Dify 工作流设计和插件开发。

### 核心特性

- **模块化设计**: 6 个独立的子 SKILL，覆盖从开发到部署的完整流程
- **渐进式披露**: 从概览到细节，按需深入，优化上下文使用
- **实用导向**: 提供开箱即用的模板、工具和最佳实践
- **持续更新**: 跟踪 Dify 官方版本，保持内容的时效性

## 快速开始

### 1. 浏览总 SKILL

```bash
# 查看总入口 SKILL
cat SKILL.md
```

总 SKILL 提供了 Dify 的概览和所有子 SKILL 的索引，帮助你快速找到需要的内容。

### 2. 选择子 SKILL

根据你的需求选择相应的子 SKILL:

- **工作流设计**: [01-workflow/SKILL.md](./01-workflow/SKILL.md)
- **插件开发**: [02-plugin/SKILL.md](./02-plugin/SKILL.md)
- **性能优化**: [03-performance/SKILL.md](./03-performance/SKILL.md)
- **安全和部署**: [04-security/SKILL.md](./04-security/SKILL.md)
- **集成和扩展**: [05-integration/SKILL.md](./05-integration/SKILL.md)
- **参考资料**: [06-reference/SKILL.md](./06-reference/SKILL.md)

### 3. 使用模板

每个子 SKILL 都提供了可直接使用的模板:

```bash
# 工作流模板
01-workflow/templates/simple-qa.yml
01-workflow/templates/rag-assistant.yml
01-workflow/templates/agent-tools.yml

# 插件模板
02-plugin/templates/tool-template/
02-plugin/templates/model-template/
02-plugin/templates/datasource-template/
```

## 目录结构

```
dify-skills/
├── SKILL.md                    # 总入口 SKILL
├── ARCHITECTURE.md             # 架构设计文档
├── README.md                   # 本文件
│
├── 01-workflow/                # 工作流设计 SKILL
│   ├── SKILL.md
│   ├── nodes/                 # 节点类型详解
│   ├── patterns/              # 设计模式
│   ├── templates/             # 工作流模板
│   └── best-practices.md
│
├── 02-plugin/                  # 插件开发 SKILL
│   ├── SKILL.md
│   ├── types/                 # 插件类型
│   ├── development/           # 开发指南
│   ├── templates/             # 插件模板
│   ├── testing/               # 测试指南
│   └── best-practices.md
│
├── 03-performance/             # 性能优化 SKILL
│   ├── SKILL.md
│   ├── workflow-optimization/
│   ├── llm-optimization/
│   ├── plugin-optimization/
│   └── monitoring.md
│
├── 04-security/                # 安全和部署 SKILL
│   ├── SKILL.md
│   ├── security/
│   ├── deployment/
│   ├── operations/
│   └── best-practices.md
│
├── 05-integration/             # 集成和扩展 SKILL
│   ├── SKILL.md
│   ├── api/
│   ├── mcp/
│   ├── external-services/
│   └── examples/
│
└── 06-reference/               # 参考资料 SKILL
    ├── SKILL.md
    ├── dsl-reference.md
    ├── api-reference.md
    ├── sdk-reference.md
    ├── cli-reference.md
    ├── troubleshooting.md
    └── resources.md
```

## 使用场景

### 场景 1: 新手入门

如果你是 Dify 新手，建议按以下顺序学习:

1. 阅读 [SKILL.md](./SKILL.md) 了解 Dify 全貌
2. 学习 [01-workflow/SKILL.md](./01-workflow/SKILL.md) 掌握工作流基础
3. 使用模板创建第一个工作流
4. 根据需要深入学习其他子 SKILL

### 场景 2: 插件开发

如果你需要开发自定义插件:

1. 阅读 [02-plugin/SKILL.md](./02-plugin/SKILL.md) 了解插件体系
2. 选择合适的插件类型模板
3. 参考开发指南实现功能
4. 使用测试指南验证插件

### 场景 3: 性能优化

如果你需要优化现有工作流或插件:

1. 阅读 [03-performance/SKILL.md](./03-performance/SKILL.md) 了解优化策略
2. 分析当前的性能瓶颈
3. 应用相应的优化技术
4. 使用监控工具验证效果

### 场景 4: 生产部署

如果你需要将应用部署到生产环境:

1. 阅读 [04-security/SKILL.md](./04-security/SKILL.md) 了解安全和部署
2. 选择合适的部署方式
3. 配置监控和日志
4. 建立运维流程

## 设计理念

### 模块化

每个子 SKILL 都是独立完整的，可以单独使用。你不需要阅读所有内容，只需要选择你需要的部分。

### 渐进式披露

内容分为三个层次:

- **Level 1 - 总 SKILL**: 概览和导航 (~500 行)
- **Level 2 - 子 SKILL**: 核心知识和常用操作 (~300-500 行)
- **Level 3 - 详细文档**: 特定主题的深入讲解 (~200-300 行)

这种设计可以最小化单次加载的内容量，优化上下文使用。

### 实用导向

所有内容都以实用为导向:

- 提供可直接使用的代码和配置
- 包含真实的案例和最佳实践
- 所有代码示例都经过测试验证

### 持续演进

SKILL 会持续更新:

- 跟踪 Dify 官方版本更新
- 定期更新最佳实践
- 收集社区反馈并改进

## 贡献指南

我们欢迎社区贡献！你可以通过以下方式参与:

### 报告问题

如果你发现文档错误或有改进建议，请提交 Issue。

### 提交改进

1. Fork 本仓库
2. 创建你的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交你的改动 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启一个 Pull Request

### 贡献内容

你可以贡献:

- 新的工作流模板
- 新的插件模板
- 最佳实践案例
- 文档改进和翻译
- 工具和脚本

## 版本历史

### v1.0.0 (2026-03-04)

- 初始版本发布
- 完成架构设计
- 创建总 SKILL 和目录结构
- 规划 6 个子 SKILL

### 未来计划

- v1.1.0: 完成核心子 SKILL 的开发
- v1.2.0: 增加高级主题和更多模板
- v2.0.0: 提供 SKILL 管理工具和在线浏览器

## 相关资源

### 官方资源

- [Dify 官方文档](https://docs.dify.ai/)
- [Dify GitHub](https://github.com/langgenius/dify)
- [Dify 插件市场](https://marketplace.dify.ai/)
- [Dify 官方博客](https://dify.ai/blog/)

### 社区资源

- [Discord 社区](https://discord.gg/FngNHpbcY7)
- [Reddit 社区](https://reddit.com/r/difyai)
- [GitHub Discussions](https://github.com/langgenius/dify/discussions)

### 学习资源

- [Dify 快速入门](https://docs.dify.ai/en/use-dify/getting-started/quick-start)
- [工作流教程](https://docs.dify.ai/en/guides/workflow/)
- [插件开发指南](https://docs.dify.ai/en/develop-plugin/)

## 许可证

本项目基于 MIT 许可证开源。详见 [LICENSE](LICENSE) 文件。

## 致谢

感谢 Dify 团队和社区的贡献，感谢所有参与 SKILL 开发和改进的贡献者。

---

**开始你的 Dify 开发之旅吧！**

如有任何问题或建议，欢迎通过 Issue 或 Discussion 与我们交流。

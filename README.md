# 智能体工程 Agent Engineering

> **探索智能体应用开发的最佳实践与实战经验**

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-sa/4.0/)
[![Status](https://img.shields.io/badge/status-active-success.svg)]()
[![Articles](https://img.shields.io/badge/articles-6-blue.svg)]()

---

## 📖 仓库介绍

**智能体工程 Agent Engineering** 是一个专注于智能体（AI Agent）应用开发的实战经验仓库。

在这里，我们分享：
- 🎯 智能体架构设计心得
- 🛠️ 工程化实践经验
- 📊 性能优化技巧
- 🔧 工具链建设
- 💡 实战案例解析

**目标读者**：
- 正在或计划开发智能体应用的开发者
- 对 AI Agent 工程化感兴趣的技术人员
- 想要提升智能体应用质量的团队

---

## 📚 文章目录

### 📍 第 1 篇：80 分钟搭建 AI 秘书团队：Harness Engineering 完整实施指南

**目录**：`01-harness-engineering-guide/`  
**发布时间**：2026-04-21（2026-04-23 更新）  
**版本**：v1.1  
**阅读时间**：约 30 分钟  
**状态**：✅ 已完成  
**标签**：Agent, OpenClaw, Harness, Workflow, Token

**核心内容**：
- 4 个阶段、15 个脚本、5 个工作流模板
- 系统自愈率>80%
- 智能路由、并行编排、可观测性
- 图片 Token 检查和自动压缩（2026-04-23 新增）

[📖 阅读全文](articles/01-harness-engineering-guide/harness-engineering-implementation-guide.md)

---

### 📍 第 2 篇：Harness Engineering 工作流引擎使用教程

**目录**：`02-workflow-tutorial/`  
**发布时间**：2026-04-21  
**阅读时间**：约 20 分钟  
**状态**：✅ 已完成  
**标签**：Workflow, YAML, 自动化

**核心内容**：
- 工作流引擎架构解析
- 节点类型与执行流程
- 实战案例分享
- 5 个开箱即用模板

[📖 阅读全文](articles/02-workflow-tutorial/workflow-engine-tutorial.md)

---

### 📍 第 3 篇：LLM 上下文管理最佳实践：三级预警应对指南

**目录**：`03-llm-context-warning/`  
**发布时间**：2026-04-22（2026-04-23 更新）  
**版本**：v1.1  
**阅读时间**：约 8 分钟  
**状态**：✅ 已完成  
**标签**：LLM, Token, 上下文管理, 预警系统

**核心内容**：
- 三级预警系统（黄/橙/红）
- 分级应对策略
- 常见问题与解决方案
- 图片 + 文本 Token 超限处理（2026-04-23 新增）

[📖 阅读全文](articles/03-llm-context-warning/llm-.md)

---

### 📍 第 4 篇：OpenClaw 上下文监控中间件：防止 LLM 失忆的三级预警系统

**目录**：`04-context-middleware/`  
**发布时间**：2026-04-22  
**阅读时间**：约 3 分钟  
**状态**：✅ 已完成  
**标签**：OpenClaw, Middleware, 监控

**核心内容**：
- 工程实践分享
- 完整实施指南
- 可复用代码示例

[📖 阅读全文](articles/04-context-middleware/openclaw-llm-.md)

---

### 📍 第 5 篇：多 Agent 协同 CI/CD 流水线实战：基于 OpenClaw 的智能自动化部署方案

**目录**：`05-multiagent-cicd/`  
**发布时间**：2026-04-22  
**阅读时间**：约 25 分钟  
**状态**：✅ 已完成  
**标签**：CI/CD, Agent, OpenClaw, 自动化, DevOps

**核心内容**：
- 8 个 Agent 角色详解与协同模式
- 基于 OpenClaw 的完整实现方案
- 最佳实践与性能优化技巧

[📖 阅读全文](articles/05-multiagent-cicd/openclaw-multiagent-cicd-practice.md)

---

### 📍 第 6 篇：从文档约束到 Skill 工具化：Project Context 多 Agent 上下文传递系统实施实践

**目录**：`06-project-context-skill/`  
**创建时间**：2026-04-23  
**状态**：🔍 观察期（2026-04-23 至 2026-04-25）  
**标签**：#多 Agent 协作 #上下文传递 #项目缓存 #Skill 工具化 #代码固化对比

**核心内容**：
- Project Context Skill 设计与实现
- 多 Agent 上下文传递方案
- 代码固化对比分析
- 实施效果评估

[📖 阅读全文](articles/06-project-context-skill/README.md)

---

## 📚 完整文章列表

| 序号 | 标题 | 发布 | 更新 | 状态 | 标签 |
|------|------|------|------|------|
| 1 | Harness Engineering 完整实施指南 | 2026-04-21 | 2026-04-23 | ✅ | Harness, Token, Workflow |
| 2 | 工作流引擎使用教程 | 2026-04-21 | - | ✅ | Workflow, YAML |
| 3 | LLM 上下文管理最佳实践 | 2026-04-22 | 2026-04-23 | ✅ | LLM, Token, 预警 |
| 4 | 上下文监控中间件 | 2026-04-22 | - | ✅ | Middleware, 监控 |
| 5 | 多 Agent 协同 CI/CD 流水线实战 | 2026-04-22 | - | ✅ | CI/CD, DevOps |
| 6 | Project Context 多 Agent 上下文传递 | 2026-04-23 | - | 🔍 | 上下文, 缓存 |

---

## 🎯 快速开始

### 阅读顺序建议

**入门系列**：
1. 📍 第 1 篇：了解 Harness Engineering 整体架构
2. 📍 第 2 篇：掌握工作流引擎使用方法
3. 📍 第 3 篇：学习上下文管理最佳实践

**进阶系列**：
4. 📍 第 4 篇：深入中间件实现细节
5. 📍 第 5 篇：复杂场景 - 多 Agent CI/CD
6. 📍 第 6 篇：最新实践 - Project Context Skill

### 技术栈

- **平台**：OpenClaw
- **语言**：JavaScript / Shell / Python
- **工具**：sharp, sqlite3, cron
- **部署**：GitHub Actions, Docker

---

## 📝 更新日志

### 2026-04-23
- ✅ 新增第 6 篇文章：Project Context 多 Agent 上下文传递
- ✅ 更新第 1 篇和第 3 篇，新增图片 Token 检查功能
- ✅ 重命名所有目录，统一编号格式
- ✅ 更新 README 文章目录

### 2026-04-22
- ✅ 发布第 3、4、5 篇文章
- ✅ 完善目录结构

### 2026-04-21
- ✅ 发布第 1、2 篇文章
- ✅ 初始化仓库结构

---

## 🤝 贡献指南

欢迎贡献！你可以通过以下方式参与：

1. **提交 Issue**：发现问题或提出建议
2. **Pull Request**：修复 Bug 或新增内容
3. **分享经验**：分享你的 Agent 工程实践
4. **翻译文档**：帮助翻译成其他语言

**提交规范**：
- 文章请使用 Markdown 格式
- 代码示例请保证可运行
- 配图建议使用 Mermaid 代码（GitHub 原生支持）

---

## 📄 许可证

本仓库所有文章采用 **CC BY-NC-SA 4.0** 许可证：
- ✅ 可以分享、复制
- ✅ 可以改编、修改
- ❌ 不得用于商业用途
- ❌ 必须署名
- ❌ 相同方式共享

代码示例采用 **MIT** 许可证。

---

## 📱 关注我

**微信公众号**: 智能体开发

专注于分享：
- AI Agent 开发与自动化
- Harness Engineering 实战
- OpenClaw 技术应用
- 编程效率提升

![关注公众号](https://github.com/subaochen/agent-engineering/raw/main/assets/wechat-qr.jpg)

> **👆 长按二维码，关注"智能体开发"**

*扫码关注，获取最新文章和技术干货*

---

## 🔗 相关链接

- **GitHub 仓库**：https://github.com/subaochen/agent-engineering
- **OpenClaw 官网**：https://openclaw.ai
- **Issue 反馈**：https://github.com/subaochen/agent-engineering/issues
- **微信交流群**：扫码加入（见公众号）

---

*最后更新：2026-04-23*

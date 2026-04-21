# Harness Engineering 工作流引擎使用教程

> **摘要**：手把手教你使用工作流引擎，从 YAML 定义到执行监控，5 个模板开箱即用。包含完整示例和常见问题解答。

---

## 📋 前言

在上一篇文章中，我们分享了 [Harness Engineering 完整实施指南](./harness-engineering-implementation-guide.md)，介绍了如何从 0 搭建多 Agent 协作体系。

很多读者问：**工作流引擎到底怎么用？复杂吗？我能直接套用吗？**

这篇文章就是答案。

**你将学到**：
- ✅ 工作流引擎的基本概念
- ✅ 如何定义工作流（YAML 格式）
- ✅ 如何执行和监控工作流
- ✅ 5 个开箱即用的模板
- ✅ 实战演示（含执行日志）
- ✅ 常见问题解答

**前提条件**：
- 已部署 OpenClaw 环境
- 了解基本的 Bash 命令
- 有秘书长 + 子代理架构（或类似多 Agent 系统）

---

## 🎯 一、工作流引擎是什么？

### 1.1 核心概念

**工作流（Workflow）**：一系列按顺序执行的任务步骤。

**工作流引擎**：解析工作流定义、调度执行、管理状态的运行时系统。

**类比理解**：
```
工作流 = 菜谱（定义步骤）
工作流引擎 = 厨师（执行步骤）
数据库 = 记事本（记录进度）
```

### 1.2 适用场景

**适合用工作流的场景**：
- ✅ 固定流程的重复性任务
- ✅ 需要多角色协作的任务
- ✅ 有明确先后顺序的任务
- ✅ 需要追踪进度的任务

**典型例子**：
- 公众号文章发布（抓取→润色→发布）
- 会议安排（查日程→确认→发邀请）
- 周报生成（收集→整理→汇总）
- 市场调研（多路调研→对比→报告）

**不适合的场景**：
- ❌ 一次性临时任务
- ❌ 流程不确定的探索性任务
- ❌ 对延迟极度敏感的任务

### 1.3 架构概览

```
┌─────────────────────────────────────────┐
│   YAML 工作流定义文件                    │
│   - 名称、步骤、超时、错误处理           │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│   工作流引擎 (workflow-engine.sh)        │
│   - 解析 YAML                            │
│   - 创建数据库记录                       │
│   - 按顺序执行步骤                       │
│   - 更新状态                             │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│   SQLite 数据库                          │
│   - workflows 表（工作流实例）           │
│   - workflow_steps 表（步骤详情）        │
│   - agent_stats 表（性能统计）           │
└─────────────────────────────────────────┘
```

---

## 🚀 二、快速开始

### 2.1 查看已有工作流模板

```bash
# 进入工作目录
cd /home/sbc/.openclaw/workspace

# 查看可用模板
ls -la workflows/templates/
```

**输出**：
```
-rw-rw-r-- 1 sbc sbc 891 4 月 21 10:08 market-research.yaml
-rw-rw-r-- 1 sbc sbc 929 4 月 21 10:08 meeting-schedule.yaml
-rw-rw-r-- 1 sbc sbc 942 4 月 21 10:09 multi-research.yaml
-rw-rw-r-- 1 sbc sbc 807 4 月 21 10:08 wechat-publish.yaml
-rw-rw-r-- 1 sbc sbc 849 4 月 21 10:09 weekly-report.yaml
```

**5 个模板说明**：
| 模板 | 用途 | 步骤数 | 预计耗时 |
|------|------|--------|----------|
| `wechat-publish.yaml` | 公众号文章发布 | 3 | 30 分钟 |
| `meeting-schedule.yaml` | 会议安排 | 4 | 45 分钟 |
| `weekly-report.yaml` | 周报生成 | 3 | 20 分钟 |
| `market-research.yaml` | 市场调研 | 3（含并行） | 60 分钟 |
| `multi-research.yaml` | 多路调研 | 3（并行） | 30 分钟 |

---

### 2.2 执行第一个工作流

**示例**：执行公众号文章发布工作流

```bash
# 执行工作流
./workflow-engine.sh run workflows/templates/wechat-publish.yaml
```

**执行输出**：
```
=== 工作流引擎启动 2026-04-21T10:13:38+08:00 ===
工作流：公众号文章发布
步骤数：3
✅ 工作流记录创建：WF-20260421-101338-7721

📋 工作流已创建：WF-20260421-101338-7721
   名称：公众号文章发布
   步骤：3

🚀 开始执行工作流...

🔧 执行步骤：step1
   子代理：qingbao
   任务："抓取公众号文章并提取核心内容"
   状态：执行中...
   ✅ 完成

🔧 执行步骤：step2
   子代理：wenan
   任务："润色文案，优化标题和摘要，生成封面图建议"
   状态：执行中...
   ✅ 完成

🔧 执行步骤：step3
   子代理：秘书长
   任务："发布到微信公众号，记录发布结果"
   状态：执行中...
   ✅ 完成

=== 工作流执行完成 ===

📊 执行结果：
   工作流 ID: WF-20260421-101338-7721
   状态：completed
   完成步骤：3/3
```

**解读**：
1. 引擎解析 YAML 文件，提取工作流名称和步骤数
2. 生成唯一工作流 ID（`WF-YYYYMMDD-HHMMSS-随机数`）
3. 在数据库中创建记录
4. 按顺序执行每个步骤（step1 → step2 → step3）
5. 所有步骤完成后，更新工作流状态为 `completed`

---

### 2.3 查看工作流状态

```bash
# 查看特定工作流状态
./workflow-engine.sh status WF-20260421-101338-7721
```

**输出**：
```
WF-20260421-101338-7721|公众号文章发布|completed|3|3|2026-04-21 02:13:38
```

**字段说明**（从左到右）：
- 工作流 ID
- 工作流名称
- 状态（pending/running/completed/failed）
- 当前步骤
- 总步骤数
- 创建时间

---

### 2.4 查看所有工作流历史

```bash
# 列出最近 10 个工作流
./workflow-engine.sh list
```

**输出示例**：
```
WF-20260421-101338-7721|公众号文章发布|completed|2026-04-21 10:13:38
WF-20260421-095000-3456|会议安排|completed|2026-04-21 09:50:00
WF-20260420-180000-7890|周报生成|failed|2026-04-20 18:00:00
```

---

## 📝 三、自定义工作流

### 3.1 YAML 格式详解

**完整结构**：
```yaml
workflow:
  name: "工作流名称"
  description: "工作流描述"
  version: "1.0"
  
  steps:
    - id: step1
      agent: qingbao
      task: "任务描述"
      timeout_minutes: 10
      on_fail: retry
      
    - id: step2
      agent: wenan
      task: "任务描述"
      timeout_minutes: 15
      on_fail: notify
      
  output:
    - step3.output
    - logs/workflow-{{workflow_id}}.md
```

**字段说明**：

| 字段 | 必填 | 说明 | 示例 |
|------|------|------|------|
| `name` | ✅ | 工作流名称 | `"公众号文章发布"` |
| `description` | ❌ | 工作流描述 | `"抓取→润色→发布"` |
| `version` | ❌ | 版本号 | `"1.0"` |
| `steps` | ✅ | 步骤列表 | 见下方 |
| `output` | ❌ | 输出文件列表 | 支持变量 `{{workflow_id}}` |

**步骤字段**：

| 字段 | 必填 | 说明 | 可选值 |
|------|------|------|--------|
| `id` | ✅ | 步骤唯一标识 | `step1`, `step2` |
| `agent` | ✅ | 执行的子代理 | `qingbao`, `wenan`, `jiaoxue`, `秘书长` |
| `task` | ✅ | 任务描述 | 字符串 |
| `timeout_minutes` | ❌ | 超时时间（分钟） | 数字 |
| `on_fail` | ❌ | 失败处理策略 | `retry`, `notify`, `abort`, `continue` |

---

### 3.2 实战：创建一个简单工作流

**需求**：创建一个"每日新闻摘要"工作流

**流程**：
1. qingbao 收集新闻（10 分钟）
2. wenan 整理摘要（15 分钟）
3. 秘书长发送到微信群（5 分钟）

**创建文件**：`workflows/templates/daily-news.yaml`

```yaml
workflow:
  name: "每日新闻摘要"
  description: "收集新闻 → 整理摘要 → 发送到微信群"
  version: "1.0"
  
  steps:
    - id: step1
      agent: qingbao
      task: "收集今日科技新闻，提取 Top 10 热点"
      timeout_minutes: 10
      on_fail: retry
      
    - id: step2
      agent: wenan
      task: "整理新闻摘要，每条新闻不超过 100 字"
      timeout_minutes: 15
      on_fail: retry
      
    - id: step3
      agent: 秘书长
      task: "发送到微信群，记录发送结果"
      timeout_minutes: 5
      on_fail: notify
      
  output:
    - step3.output
    - logs/daily-news-{{date}}.md
```

**执行**：
```bash
./workflow-engine.sh run workflows/templates/daily-news.yaml
```

---

### 3.3 错误处理策略

**4 种失败处理**：

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| `retry` | 自动重试（最多 2 次） | 临时错误（网络波动、API 限流） |
| `notify` | 通知秘书长，等待人工介入 | 严重错误（认证失败、权限不足） |
| `abort` | 立即终止工作流 | 致命错误（数据丢失、系统崩溃） |
| `continue` | 跳过错误，继续下一步 | 非关键步骤（日志记录、可选功能） |

**示例**：
```yaml
steps:
  - id: step1
    agent: qingbao
    task: "抓取数据"
    on_fail: retry  # 临时错误，重试
    
  - id: step2
    agent: wenan
    task: "分析数据"
    on_fail: notify  # 需要人工判断
    
  - id: step3
    agent: 秘书长
    task: "发送报告"
    on_fail: abort  # 发送失败，终止
```

---

## 🔥 四、高级用法

### 4.1 并行工作流

**使用场景**：同时调研多个竞品，然后合并结果

**工具**：`parallel-dispatch.sh`

**示例**：
```bash
# 并行分发任务
./parallel-dispatch.sh "调研 AI 教育产品" "qingbao,wenan,jiaoxue" compare

# 参数说明：
# - "调研 AI 教育产品"：任务描述
# - "qingbao,wenan,jiaoxue"：子代理列表（逗号分隔）
# - "compare"：合并策略（concat/compare/select_best）
```

**输出**：
```
=== 并行任务分发 2026-04-21T10:30:00+08:00 ===
📋 任务信息：
   父任务 ID: PT-20260421-103000-1234
   任务描述：调研 AI 教育产品
   子任务数：3
   子代理：qingbao,wenan,jiaoxue
   合并策略：compare

🚀 分发子任务...
   子任务 1: qingbao → 调研 AI 教育产品 [子任务 1/3]
   子任务 2: wenan → 调研 AI 教育产品 [子任务 2/3]
   子任务 3: jiaoxue → 调研 AI 教育产品 [子任务 3/3]

✅ 所有子任务已分发
✅ 所有子任务完成

📊 合并结果：
### qingbao
从技术角度分析，AI 教育产品核心是...

### wenan
从市场角度看，用户需求集中在...

### jiaoxue
从教学应用看，落地场景包括...
```

---

### 4.2 智能路由

**使用场景**：不确定任务应该派给哪个子代理？让智能路由帮你决定！

**工具**：`smart-router.sh`

**示例 1**：基于任务类型路由
```bash
./smart-router.sh route "调研竞品 A 的功能特点"
```

**输出**：
```
=== 智能路由决策 2026-04-21T10:35:00+08:00 ===
📋 任务分析：
   描述：调研竞品 A 的功能特点
   类型：调研

✅ 基于任务类型推荐：qingbao

📊 qingbao 统计：
   总任务：50
   成功率：95%
   平均耗时：8 分钟

qingbao
```

**示例 2**：查看子代理统计
```bash
./smart-router.sh stats
```

**输出**：
```
📊 qingbao 统计：
   总任务：50
   成功率：95%
   平均耗时：8 分钟

📊 wenan 统计：
   总任务：45
   成功率：92%
   平均耗时：12 分钟

📊 jiaoxue 统计：
   总任务：60
   成功率：98%
   平均耗时：5 分钟

📊 self 统计：
   总任务：100
   成功率：99%
   平均耗时：3 分钟
```

**示例 3**：手动指定子代理
```bash
./smart-router.sh route "写一份产品文案" wenan
# 输出：wenan（直接使用指定子代理）
```

---

### 4.3 数据库查询

**直接查询 SQLite 数据库**：

```bash
# 查询所有已完成的工作流
sqlite3 /home/sbc/.openclaw/workspace/data/workflows.db \
  "SELECT id, name, created_at FROM workflows WHERE status='completed';"

# 查询某个子代理的所有任务
sqlite3 /home/sbc/.openclaw/workspace/data/workflows.db \
  "SELECT * FROM workflow_steps WHERE agent='qingbao';"

# 查询工作流平均耗时
sqlite3 /home/sbc/.openclaw/workspace/data/workflows.db \
  "SELECT AVG(julianday(completed_at) - julianday(created_at)) FROM workflows WHERE status='completed';"
```

---

## 🎯 五、5 个模板详解

### 模板 1：公众号文章发布

**文件**：`wechat-publish.yaml`

**流程**：
```
qingbao（抓取） → wenan（润色） → 秘书长（发布）
```

**适用场景**：
- 微信公众号文章发布
- 博客文章发布
- 新闻稿发布

**自定义建议**：
- 修改 step1 的任务描述，适配不同来源（知乎/微博/网站）
- 在 step2 增加"生成封面图建议"
- 在 step3 修改发布渠道（公众号/知乎/博客）

---

### 模板 2：市场调研

**文件**：`market-research.yaml`

**流程**：
```
并行 3 路调研 → wenan（汇总对比） → 秘书长（生成报告）
```

**适用场景**：
- 竞品分析
- 市场调研
- 产品对比

**自定义建议**：
- 修改竞品数量（增加/减少并行路数）
- 修改对比维度（功能/价格/用户体验）
- 增加 SWOT 分析步骤

---

### 模板 3：会议安排

**文件**：`meeting-schedule.yaml`

**流程**：
```
jiaoxue（查日程） → 秘书长（确认时间） → jiaoxue（发邀请） → 秘书长（记录）
```

**适用场景**：
- 团队会议
- 客户会议
- 面试安排

**自定义建议**：
- 增加"预订会议室"步骤
- 增加"准备会议材料"步骤
- 修改确认方式（飞书/邮件/电话）

---

### 模板 4：周报生成

**文件**：`weekly-report.yaml`

**流程**：
```
qingbao（收集数据） → wenan（撰写总结） → 秘书长（汇总发送）
```

**适用场景**：
- 团队周报
- 项目周报
- 个人周报

**自定义建议**：
- 增加"收集下周计划"步骤
- 增加"领导审核"步骤
- 修改报告格式（Markdown/Word/邮件）

---

### 模板 5：多路调研

**文件**：`multi-research.yaml`

**流程**：
```
并行 3 路（技术/市场/教学） → wenan（对比分析） → 秘书长（综合评估）
```

**适用场景**：
- 多角度调研
- 跨部门协作
- 综合评估

**自定义建议**：
- 修改调研角度（技术/市场/运营/财务）
- 修改合并策略（concat/compare/select_best）
- 增加"专家审核"步骤

---

## 🐛 六、常见问题（FAQ）

### Q1: 工作流执行失败怎么办？

**排查步骤**：
```bash
# 1. 查看工作流状态
./workflow-engine.sh status <workflow_id>

# 2. 查看失败步骤
sqlite3 /home/sbc/.openclaw/workspace/data/workflows.db \
  "SELECT id, agent, task, status, output FROM workflow_steps WHERE status='failed';"

# 3. 查看日志
tail -50 logs/workflow.log

# 4. 重试工作流
./workflow-engine.sh run workflows/templates/xxx.yaml
```

**常见原因**：
- 子代理不可用（检查心跳）
- 超时（增加 `timeout_minutes`）
- API 错误（检查网络和配额）

---

### Q2: 如何修改已有工作流？

**方法 1**：直接编辑 YAML 文件
```bash
vim workflows/templates/wechat-publish.yaml
```

**方法 2**：复制后修改（推荐）
```bash
cp workflows/templates/wechat-publish.yaml workflows/templates/wechat-publish-v2.yaml
# 编辑新文件
```

**注意**：修改后版本号 +1（`version: "2.0"`）

---

### Q3: 工作流能嵌套吗？

**可以**，但需要手动实现：

```yaml
# 在主工作流中调用子工作流
steps:
  - id: step1
    agent: 秘书长
    task: "执行子工作流：./workflow-engine.sh run sub-workflow.yaml"
```

**更好的方式**：使用并行工作流引擎（待实现）

---

### Q4: 如何监控工作流进度？

**方法 1**：查看数据库
```bash
sqlite3 /home/sbc/.openclaw/workspace/data/workflows.db \
  "SELECT name, status, current_step, total_steps FROM workflows ORDER BY created_at DESC;"
```

**方法 2**：使用飞书多维表格看板
- URL：https://my.feishu.cn/base/FfFybGSIRaQRQksfQeZcQrU4nVc
- 自动同步数据库状态

**方法 3**：定时任务监控
```bash
# 每小时检查一次
0 * * * * /home/sbc/.openclaw/workspace/scripts/workflow-engine.sh list >> logs/workflow-monitor.log
```

---

### Q5: 工作流执行太慢怎么办？

**优化建议**：

1. **并行化**：将串行步骤改为并行
   ```bash
   # 使用 parallel-dispatch.sh
   ./parallel-dispatch.sh "任务描述" "qingbao,wenan,jiaoxue" concat
   ```

2. **减少超时时间**：
   ```yaml
   steps:
     - id: step1
       timeout_minutes: 5  # 从 10 分钟改为 5 分钟
   ```

3. **优化子代理性能**：
   - 查看统计：`./smart-router.sh stats`
   - 选择更快的子代理

4. **拆分工作流**：将大工作流拆分为多个小工作流

---

### Q6: 如何备份工作流数据？

**备份数据库**：
```bash
# 每天备份一次
cp /home/sbc/.openclaw/workspace/data/workflows.db \
   /home/sbc/.openclaw/workspace/data/workflows.db.backup.$(date +%Y%m%d)

# 保留最近 7 天备份
find /home/sbc/.openclaw/workspace/data/ -name "*.backup.*" -mtime +7 -delete
```

**备份 YAML 模板**：
```bash
# 打包所有模板
tar -czf workflows-backup-$(date +%Y%m%d).tar.gz workflows/templates/
```

---

## 📊 七、最佳实践

### 7.1 工作流设计原则

1. **单一职责**：一个工作流只做一件事
2. **步骤精简**：每个工作流 3-5 个步骤为宜
3. **错误处理**：每个步骤都要有 `on_fail` 策略
4. **超时控制**：设置合理的 `timeout_minutes`
5. **日志记录**：输出关键信息到日志文件

### 7.2 命名规范

**工作流 ID**：
```
WF-YYYYMMDD-HHMMSS-随机数
示例：WF-20260421-101338-7721
```

**工作流名称**：
```
<场景>_<动作>
示例：wechat_publish、meeting_schedule
```

**步骤 ID**：
```
step<序号>
示例：step1、step2、step3
```

### 7.3 监控告警

**配置告警规则**：
```bash
# 工作流失败率>20% 时告警
# 工作流超时>30 分钟时告警
# 连续 3 个工作流失败时告警
```

**告警方式**：
- 飞书消息（推荐）
- 邮件
- 短信（严重故障）

---

## 🎉 八、总结

通过这篇文章，你学会了：

1. ✅ 工作流引擎的基本概念和架构
2. ✅ 如何执行已有工作流模板
3. ✅ 如何自定义工作流（YAML 格式）
4. ✅ 高级用法（并行、路由、查询）
5. ✅ 5 个模板的详细用法
6. ✅ 常见问题和最佳实践

**下一步行动**：

1. **试用模板**：执行一个工作流模板
   ```bash
   ./workflow-engine.sh run workflows/templates/wechat-publish.yaml
   ```

2. **自定义工作流**：创建一个适合你的工作流
   ```bash
   vim workflows/templates/my-first-workflow.yaml
   ```

3. **分享经验**：在评论区分享你的使用心得

---

## 📚 九、延伸阅读

- [Harness Engineering 完整实施指南](./harness-engineering-implementation-guide.md)
- OpenClaw 官方文档：https://docs.openclaw.ai
- ACP 协议规范：https://github.com/openclaw/acp

---

## 💬 十、互动环节

**问题征集**：
- 你遇到了什么问题？
- 你想实现什么工作流？
- 你有什么优化建议？

欢迎在评论区留言，我们会逐一回复！

**模板征集**：
- 如果你创建了有用的工作流模板
- 欢迎提交 PR 到我们的模板库
- 让更多用户受益

---

**作者**：[你的名字]  
**编辑**：秘书长 AI 团队  
**发布日期**：2026-04-21  
**版本**：v1.0

---

*本文采用 CC BY-NC-SA 4.0 许可证，转载请注明出处。*

*配套代码和模板：https://github.com/your-repo/harness-engineering*

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

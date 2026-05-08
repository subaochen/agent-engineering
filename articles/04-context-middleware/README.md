# OpenClaw 上下文监控中间件：防止 LLM"失忆"的三级预警系统

> 🧠 **实时监控对话上下文使用量，在撑爆之前主动预警**

---

## 😱 问题：为什么你的 AI 会"失忆"？

你有没有遇到过这种情况：

- 聊着聊着，AI 突然忘记了前面说过的话
- 对话进行到一半，AI 开始胡言乱语
- 明明刚才还在讨论的话题，AI 突然说"我不太理解"

**这不是 AI 变笨了，是上下文被撑爆了！**

### LLM 的致命限制

所有大语言模型都有**固定的上下文窗口**：

| 模型 | 上下文窗口 | 约等于 |
|------|-----------|--------|
| GPT-4 Turbo | 128K tokens | 10 万汉字 |
| Claude 3 | 200K tokens | 15 万汉字 |
| Qwen Max | 32K tokens | 2.5 万汉字 |

**但对话是无限增长的！**

当对话 token 数 > 上下文窗口时，会发生三种灾难：

1. **😱 静默截断** - 系统偷偷丢掉最早的消息，你完全不知道
2. **💥 API 报错** - 返回 400：`context_length_exceeded`
3. **🧠 AI 失忆** - 关键信息被截断后，回答质量断崖式下降

**最可怕的是第 1 种，因为你完全感知不到发生了什么。**

---

## 🎯 解决方案：上下文监控中间件

我们在 OpenClaw 中实现了一个**运行时中间件**，在消息送入 LLM 之前就检查 token 使用量，提前预警。

### 核心设计原则

| 原则 | 说明 |
|------|------|
| ✅ **运行时拦截** | 在消息处理前检查，不占用对话 token |
| ✅ **三级预警** | 50% 黄色 / 75% 橙色 / 90% 红色 |
| ✅ **实时通知** | 群里直接显示警告卡片 |
| ✅ **一键操作** | 压缩/归档/清空，按钮直达 |
| ✅ **冷却机制** | 5 分钟内不重复告警 |

---

## 🚀 架构设计

### 整体流程

```
用户发消息
    ↓
[运行时中间件] 检查 token 使用量
    ↓
发现 78% 了！触发 'context.warning' 事件
    ↓
[飞书监听器] 捕获事件
    ↓
构造飞书卡片消息
    ↓
📱 发送到群里（就是你看到的这条消息）
```

### 组件结构

```
openclaw-lark/
├── src/
│   ├── middleware/
│   │   ├── context-monitor.js          # 中间件核心
│   │   └── register-context-monitor.js # 注册脚本
│   ├── listeners/
│   │   └── feishu-context-warning.js   # 飞书监听器
│   └── index.js                        # 入口文件（已注册中间件）
```

---

## 🔧 技术实现

### 1. Token 估算算法

```javascript
/**
 * 简易 Token 估算器
 * 中文：1.3 chars/token
 * 英文：0.75 chars/token
 */
function estimateTokens(text) {
    if (!text) return 0;
    
    const chinese = (text.match(/[\u4e00-\u9fa5]/g) || []).length;
    const other = text.length - chinese;
    
    return Math.round(chinese / 1.3 + other * 0.75);
}
```

**为什么不用官方 tokenizer？**

- 官方 tokenizer（如 `gpt-tokenizer`）需要额外依赖
- 简易估算误差在 10-20% 以内，对预警足够准确
- 性能更好（无需加载词表）

**如需精确计数，可替换为：**
```javascript
const { encode } = require('gpt-tokenizer');
function accurateTokenCount(text) {
    return encode(text).length;
}
```

---

### 2. 中间件核心类

```javascript
class ContextMonitorMiddleware {
    constructor(config = {}) {
        this.config = {
            contextWindow: 128000,
            thresholds: {
                yellow: 0.50,
                orange: 0.75,
                red: 0.90,
            },
            cooldownMs: 300000,  // 5 分钟
        };
        this.lastWarningTime = new Map();
        this.eventListeners = new Map();
    }
    
    // 中间件主函数
    async preProcess(ctx, next) {
        const analysis = this.analyzeContext(ctx);
        const { usage } = analysis;
        
        // 确定预警级别
        let level = null;
        if (usage >= 0.90) level = 'red';
        else if (usage >= 0.75) level = 'orange';
        else if (usage >= 0.50) level = 'yellow';
        
        // 触发预警（带冷却检查）
        if (level) {
            const now = Date.now();
            const lastWarning = this.lastWarningTime.get(ctx.conversation.id) || 0;
            
            if (now - lastWarning >= 300000) {
                this.lastWarningTime.set(ctx.conversation.id, now);
                await this.trigger(level, analysis, ctx);
            }
        }
        
        await next();  // 继续处理消息
    }
}
```

---

### 3. 飞书卡片构造

```javascript
function createWarningCard(data) {
    const { level, usage, totalTokens, remaining } = data;
    
    return {
        msg_type: 'interactive',
        card: {
            header: {
                template: level === 'red' ? 'red' : 'orange',
                title: { content: `${emoji} 上下文${level === 'red' ? '告' : '预警'}` }
            },
            elements: [
                {
                    tag: 'div',
                    text: {
                        content: `已用：**${totalTokens.toLocaleString()}** tokens\n` +
                               `剩余：**${remaining.toLocaleString()}** tokens`
                    }
                },
                {
                    tag: 'action',
                    actions: [
                        { tag: 'button', text: '🗜️ 压缩对话', value: '/summarize' },
                        { tag: 'button', text: '📦 归档会话', value: '/archive' },
                        { tag: 'button', text: '🧹 清空上下文', value: '/clear' }
                    ]
                }
            ]
        }
    };
}
```

---

## 📊 预警级别详解

### 🟡 黄色预警（50%）

**触发条件**：上下文使用量 ≥ 50%

**动作**：
- 记录日志
- 后台准备压缩
- **不打扰用户**

**建议操作**：
- 开始考虑归档或压缩
- 检查是否有冗余历史

---

### 🟠 橙色预警（75%）

**触发条件**：上下文使用量 ≥ 75%

**动作**：
- ✅ **发送飞书卡片到群里**
- 显示详细统计
- 提供操作按钮

**卡片示例**：
```
┌────────────────────────────────────────────────┐
│ 🟠 上下文预警                                  │
├────────────────────────────────────────────────┤
│ 当前使用：78.3%      紧急程度：注意            │
│ Token 统计：已用 100,234 / 剩余 27,766         │
├────────────────────────────────────────────────┤
│ [🗜️ 压缩对话] [📦 归档会话] [🧹 清空上下文]   │
└────────────────────────────────────────────────┘
```

**建议操作**：
- 点击"压缩对话"按钮
- 或"归档会话"保存历史
- 避免继续长对话

---

### 🔴 红色告警（90%）

**触发条件**：上下文使用量 ≥ 90%

**动作**：
- ✅ **发送红色紧急卡片**
- 建议立即处理
- 可能影响回答质量

**建议操作**：
- **立即**点击"清空上下文"
- 或开启新会话
- 不要继续当前对话

---

## 🎁 扩展能力

### 1. 自定义监听器

```javascript
const middleware = new ContextMonitorMiddleware();

// 添加日志监听器
middleware.onWarning('orange', async (data, ctx) => {
    await fs.appendFile('./logs/warnings.log', JSON.stringify(data));
});

// 添加邮件通知监听器
middleware.onWarning('red', async (data, ctx) => {
    await sendEmail({
        to: 'admin@example.com',
        subject: `🔴 上下文告警 - ${data.conversationId}`,
    });
});

// 添加 Prometheus Metrics 监听器
middleware.onWarning('orange', (data) => {
    context_warnings_total.labels({ level: data.level }).inc();
    context_usage_histogram.observe(data.usage);
});
```

---

### 2. 自动压缩

```javascript
middleware.onWarning('orange', async (data, ctx) => {
    // 后台自动压缩历史对话
    const summary = await summarizeConversation(ctx);
    
    ctx.conversation.messages = [
        ctx.conversation.messages[0],  // 保留系统提示
        { role: 'assistant', content: `【历史摘要】\n${summary}` },
    ];
    
    await ctx.sendMessage('✅ 对话已自动压缩，保留核心上下文');
});
```

---

### 3. 自动归档

```javascript
middleware.onWarning('red', async (data, ctx) => {
    // 自动归档到 Obsidian
    await archiveToObsidian(ctx.conversation);
    
    await ctx.sendMessage(`
📦 对话已自动归档

已保存到：Obsidian/archive/${ctx.conversation.id}.md

当前上下文已清空，你可以开始新的对话了。
    `);
    
    // 清空上下文
    ctx.conversation.messages = [ctx.conversation.messages[0]];
});
```

---

## 🧪 测试结果

### 测试场景

创建模拟上下文（10000 tokens 窗口）：
- 系统提示：762 tokens
- 历史消息：6134 tokens
- 当前消息：615 tokens
- **总计**：7511 tokens（75.1%）

### 测试输出

```bash
$ node src/middleware/register-context-monitor.js

🧪 测试上下文监控中间件...

[ContextMonitor] 中间件已初始化
[ContextMonitor] 预警阈值：黄 50% / 橙 75% / 红 90%
[ContextMonitor] 触发 orange 级预警：75.1%
🟠 橙色预警：75.1%
✅ 消息处理完成

📊 上下文分析结果:
{
  totalTokens: 7511,
  maxTokens: 10000,
  usage: 0.7511,
  breakdown: { systemPrompt: 762, history: 6134, current: 615 }
}
```

**✅ 测试通过！中间件正常工作！**

---

## 📈 监控指标

### 日志指标

```bash
# 查看预警次数
grep "触发.*预警" logs/app.log | wc -l

# 查看各级别分布
grep "触发.*预警" logs/app.log | cut -d' ' -f3 | sort | uniq -c
```

### Prometheus Metrics

```prometheus
# 预警次数
context_warnings_total{level="yellow"}
context_warnings_total{level="orange"}
context_warnings_total{level="red"}

# 上下文使用量分布
context_usage_histogram_bucket

# 当前 token 数
current_context_tokens
```

---

## 💡 最佳实践

### 1. 配合自动压缩

当检测到 75% 时，后台自动压缩，用户无感知：

```javascript
middleware.onWarning('orange', async (data, ctx) => {
    if (ctx.isIdle) {  // 空闲时才压缩
        const summary = await summarizeConversation(ctx);
        ctx.conversation.compress(summary);
    }
});
```

### 2. 会话级隔离

每个会话独立记录冷却时间：

```javascript
const cooldowns = new Map();

middleware.onWarning('orange', async (data) => {
    const lastWarning = cooldowns.get(data.conversationId) || 0;
    if (Date.now() - lastWarning < 300000) return;
    
    cooldowns.set(data.conversationId, Date.now());
    // 发送警告...
});
```

### 3. 动态调整阈值

根据用户反馈动态调整：

```javascript
// 用户反馈"警告太频繁"
thresholds: {
    yellow: 0.60,  // 从 50% 提高到 60%
    orange: 0.80,  // 从 75% 提高到 80%
    red: 0.95,     // 从 90% 提高到 95%
}

// 用户反馈"警告太晚"
thresholds: {
    yellow: 0.30,  // 从 50% 降低到 30%
    orange: 0.50,  // 从 75% 降低到 50%
    red: 0.70,     // 从 90% 降低到 70%
}
```

---

## 🔍 故障排查

### 问题 1：没有收到警告卡片

**检查清单**：
- [ ] 中间件是否已注册？
- [ ] 飞书客户端是否正确初始化？
- [ ] 监听器是否注册成功？
- [ ] 群机器人是否有发送消息权限？

**调试方法**：
```bash
# 查看日志
tail -f logs/app.log | grep ContextMonitor

# 手动触发测试
node src/middleware/register-context-monitor.js
```

---

### 问题 2：警告太频繁

**解决方案**：
```javascript
// 增加冷却时间
cooldownMs: 600000,  // 从 5 分钟增加到 10 分钟

// 或提高预警阈值
thresholds: {
    yellow: 0.60,  // 从 50% 提高到 60%
    orange: 0.80,  // 从 75% 提高到 80%
}
```

---

### 问题 3：Token 估算不准确

**说明**：当前使用简易估算方法，实际 token 数可能有 10-20% 误差。

**改进方案**：
```javascript
// 使用官方 tokenizer
const { encode } = require('gpt-tokenizer');

registerContextMonitor(api, feishuClient, {
    ...config,
    tokenCounter: encode,  // 使用精确计数
});
```

---

## 📚 相关文件

| 文件 | 说明 |
|------|------|
| `src/middleware/context-monitor.js` | 中间件核心 |
| `src/listeners/feishu-context-warning.js` | 飞书监听器 |
| `src/middleware/register-context-monitor.js` | 注册脚本 |
| `CONTEXT-MONITOR-GUIDE.md` | 使用指南 |

---

## 🎯 总结

**核心价值**：
1. ✅ **提前预警** - 在撑爆之前主动提醒
2. ✅ **可视化** - 群里直接显示漂亮卡片
3. ✅ **可操作** - 一键压缩/归档/清空
4. ✅ **可扩展** - 支持自定义监听器

**技术亮点**：
- 运行时拦截，不占用对话 token
- 三级预警机制，分级处理
- 事件驱动架构，解耦监听器
- 冷却机制，避免频繁打扰

**下一步规划**：
- [ ] 历史趋势图表（Chart.js）
- [ ] 自动压缩算法优化
- [ ] 多实例数据聚合
- [ ] 自定义告警规则

---

*作者：OpenClaw 团队*  
*版本：v1.0.0*  
*最后更新：2026-04-22*  
*GitHub: https://github.com/subaochen/agent-engineering*

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

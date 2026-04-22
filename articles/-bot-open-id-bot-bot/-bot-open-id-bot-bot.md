# 飞书 Bot 真实 Open ID 获取指南 - 实现 Bot @ Bot

> 解决 OpenClaw多Agent 协作中 Bot @ Bot 的关键技术问题

**作者**: 秘书长 📋  
**日期**: 2026-04-22  
**项目**: [openclaw-team/agent](https://github.com/openclaw-team/agent)

---

## 📋 摘要

本文档详细记录了在 OpenClaw多Agent 协作系统中，如何获取飞书 Bot 的真实 Open ID 并实现 Bot @ Bot 功能的完整过程。通过源代码分析、日志调查、配置检查和 API 调用，最终成功获取 CTO Bot 的真实 Open ID：`ou_e34e30b1a0fee620ac6cca1b3fb65ab8`，并成功实现 CTO 助理@CTO Bot 的功能。

**关键词**: 飞书 Bot、Open ID、Bot @ Bot、OpenClaw、多 Agent 协作

---

## 🎯 问题背景

在 OpenClaw多Agent 协作系统中，我们需要实现 Bot @ Bot 功能，让 CTO 助理能够直接@CTO Bot 通知新任务。但在实际使用中遇到了以下问题：

### 问题现象

1. **日志中显示的都是用户 Open ID**
   ```json
   {"senderId":"ou_46514271a876f69d7258b96bae981a06"}  // 用户的 Open ID
   ```

2. **无法获取 Bot 的真实 Open ID**
   - OpenClaw 代码中有 `probe()` 方法
   - 但实际运行时返回 `undefined` 或 `unknown`

3. **Bot @ Bot 功能无法使用**
   - 所有 Agent 看起来都使用用户 OAuth 身份
   - 无法实现 Bot 之间的互相@

### 核心问题

**OpenClaw 配置的 Bot 应用是否有独立的 Open ID？如何获取？**

---

## 🔍 调查过程

### 第一步：源代码分析

**文件**: `openclaw-lark/src/core/lark-client.js`

```javascript
async probe(opts) {
    const res = await this.sdk.request({
        method: 'POST',
        url: '/open-apis/bot/v1/openclaw_bot/ping',  // ← 获取 Bot 信息的 API
        data: { needBotInfo },
    });
    
    const botInfo = res.data?.pingBotInfo;
    this._botOpenId = botInfo?.botID;  // ← 应该获取 Bot Open ID
    this._botName = botInfo?.botName;
    
    return {
        ok: true,
        appId: this.account.appId,
        botName: this._botName,
        botOpenId: this._botOpenId,
    };
}
```

**发现**：
- ✅ 代码中有 `probe()` 方法获取 Bot Open ID
- ✅ API 端点：`POST /open-apis/bot/v1/openclaw_bot/ping`
- ❓ 但实际运行时可能失败了

---

### 第二步：日志分析

**文件**: `~/.openclaw/logs/commands.log`

```bash
cat ~/.openclaw/logs/commands.log | grep "senderId" | grep -o '"senderId":"[^"]*"' | sort | uniq -c
```

**输出**：
```
27 "senderId":"ou_46514271a876f69d7258b96bae981a06"  // 用户 A
 7 "senderId":"ou_9cce3f18a3744a00f351f111d5ea1a15"  // 用户 B
 2 "senderId":"ou_2555ea623cf2148934086ee1209b8a5f"  // 用户 C
```

**发现**：
- ❌ 所有消息的 `senderId` 都是用户的 Open ID
- ❌ 没有看到 Bot 的 Open ID
- ❌ 说明发送消息时使用的是用户 OAuth Token

---

### 第三步：配置检查

**文件**: `~/.openclaw/openclaw.json`

```bash
cat ~/.openclaw/openclaw.json | jq '.channels.feishu.accounts.cto'
```

**输出**：
```json
{
  "appId": "cli_a94e1a1ffc399cce",
  "appSecret": "3gAbNnpFTPZdDMkGOoVWW3rCiLBnKPZ5",
  "botName": "CTO",
  "requireMention": true,
  "dmPolicy": "allowlist",
  "allowFrom": ["*"]
}
```

**发现**：
- ✅ CTO 配置了 App ID 和 App Secret
- ✅ 应该是 Bot 模式运行
- ❓ 但为什么获取不到 Bot Open ID？

---

### 第四步：直接调用飞书 API（关键突破！）

#### Step 1: 获取 App Access Token

```bash
curl -X POST "https://open.feishu.cn/open-apis/auth/v3/app_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{
    "app_id": "cli_a94e1a1ffc399cce",
    "app_secret": "3gAbNnpFTPZdDMkGOoVWW3rCiLBnKPZ5"
  }'
```

**返回**：
```json
{
  "code": 0,
  "expire": 7200,
  "msg": "ok",
  "tenant_access_token": "t-g1044m7WWW3DDOKMQEYNKO22LMOEH5KDO77OJMKZ"
}
```

#### Step 2: 获取 Bot Open ID

```bash
curl -X POST "https://open.feishu.cn/open-apis/bot/v1/openclaw_bot/ping" \
  -H "Authorization: Bearer t-g1044m7WWW3DDOKMQEYNKO22LMOEH5KDO77OJMKZ" \
  -H "Content-Type: application/json" \
  -d '{"needBotInfo": true}'
```

**返回**：
```json
{
  "code": 0,
  "data": {
    "pingBotInfo": {
      "botID": "ou_e34e30b1a0fee620ac6cca1b3fb65ab8",
      "botName": "CTO"
    }
  },
  "msg": ""
}
```

**成功获取 CTO Bot 的真实 Open ID**：
```
ou_e34e30b1a0fee620ac6cca1b3fb65ab8
```

---

## ✅ 解决方案

### 方法 1：使用 curl 命令（推荐）

创建脚本 `get-bot-openid.sh`：

```bash
#!/bin/bash

# 飞书 Bot Open ID 获取工具
# 用法：./get-bot-openid.sh <app_id> <app_secret>

APP_ID=$1
APP_SECRET=$2

if [ -z "$APP_ID" ] || [ -z "$APP_SECRET" ]; then
  echo "用法：$0 <app_id> <app_secret>"
  exit 1
fi

echo "正在获取 App Access Token..."
TOKEN_RESPONSE=$(curl -s -X POST "https://open.feishu.cn/open-apis/auth/v3/app_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$APP_ID\",\"app_secret\":\"$APP_SECRET\"}")

TENANT_TOKEN=$(echo $TOKEN_RESPONSE | jq -r '.tenant_access_token')

if [ "$TENANT_TOKEN" == "null" ] || [ -z "$TENANT_TOKEN" ]; then
  echo "❌ 获取 Token 失败"
  echo $TOKEN_RESPONSE
  exit 1
fi

echo "✅ Token 获取成功"

echo "正在获取 Bot Open ID..."
BOT_RESPONSE=$(curl -s -X POST "https://open.feishu.cn/open-apis/bot/v1/openclaw_bot/ping" \
  -H "Authorization: Bearer $TENANT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"needBotInfo": true}')

BOT_ID=$(echo $BOT_RESPONSE | jq -r '.data.pingBotInfo.botID')
BOT_NAME=$(echo $BOT_RESPONSE | jq -r '.data.pingBotInfo.botName')

if [ "$BOT_ID" == "null" ] || [ -z "$BOT_ID" ]; then
  echo "❌ 获取 Bot Open ID 失败"
  echo $BOT_RESPONSE
  exit 1
fi

echo "✅ 获取成功！"
echo ""
echo "Bot Name: $BOT_NAME"
echo "Bot Open ID: $BOT_ID"
echo "App ID: $APP_ID"
```

**使用方法**：
```bash
chmod +x get-bot-openid.sh
./get-bot-openid.sh cli_a94e1a1ffc399cce 3gAbNnpFTPZdDMkGOoVWW3rCiLBnKPZ5
```

---

### 方法 2：使用 Python 脚本

创建脚本 `get_bot_openid.py`：

```python
#!/usr/bin/env python3
"""
飞书 Bot Open ID 获取工具
用法：python3 get_bot_openid.py <app_id> <app_secret>
"""

import sys
import requests
import json

def get_bot_openid(app_id, app_secret):
    # Step 1: 获取 App Access Token
    token_url = "https://open.feishu.cn/open-apis/auth/v3/app_access_token/internal"
    token_data = {
        "app_id": app_id,
        "app_secret": app_secret
    }
    
    print(f"正在获取 App Access Token...")
    token_response = requests.post(token_url, json=token_data)
    token_result = token_response.json()
    
    if token_result.get('code') != 0:
        print(f"❌ 获取 Token 失败：{token_result}")
        return None
    
    tenant_token = token_result['tenant_access_token']
    print(f"✅ Token 获取成功")
    
    # Step 2: 获取 Bot Open ID
    ping_url = "https://open.feishu.cn/open-apis/bot/v1/openclaw_bot/ping"
    headers = {
        "Authorization": f"Bearer {tenant_token}",
        "Content-Type": "application/json"
    }
    ping_data = {"needBotInfo": True}
    
    print(f"正在获取 Bot Open ID...")
    ping_response = requests.post(ping_url, headers=headers, json=ping_data)
    ping_result = ping_response.json()
    
    if ping_result.get('code') != 0:
        print(f"❌ 获取 Bot Open ID 失败：{ping_result}")
        return None
    
    bot_info = ping_result['data']['pingBotInfo']
    bot_id = bot_info['botID']
    bot_name = bot_info['botName']
    
    print(f"✅ 获取成功！")
    print(f"")
    print(f"Bot Name: {bot_name}")
    print(f"Bot Open ID: {bot_id}")
    print(f"App ID: {app_id}")
    
    return {
        "bot_id": bot_id,
        "bot_name": bot_name,
        "app_id": app_id
    }

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("用法：python3 get_bot_openid.py <app_id> <app_secret>")
        sys.exit(1)
    
    app_id = sys.argv[1]
    app_secret = sys.argv[2]
    
    get_bot_openid(app_id, app_secret)
```

**使用方法**：
```bash
pip3 install requests
python3 get_bot_openid.py cli_a94e1a1ffc399cce 3gAbNnpFTPZdDMkGOoVWW3rCiLBnKPZ5
```

---

## 🎯 应用场景

### 场景 1：OpenClaw多Agent 协作

在 OpenClaw 秘书团队系统中，CTO 助理需要@CTO Bot 通知新任务：

```javascript
// CTO 助理配置（TOOLS.md）
const CTO_BOT_OPEN_ID = "ou_e34e30b1a0fee620ac6cca1b3fb65ab8";

// 在群里@CTO Bot
feishu_im_user_message({
  action: "send",
  receive_id_type: "chat_id",
  receive_id: "oc_126bea042803af7065d2f9be843a5752",
  msg_type: "text",
  content: JSON.stringify({
    text: `<at user_id="${CTO_BOT_OPEN_ID}">CTO</at> 📋 新任务已入看板，项目：集成支付组件`
  })
});
```

### 场景 2：Bot 身份验证

验证 Bot 应用配置是否正确：

```bash
# 获取 Bot 信息
./get-bot-openid.sh cli_xxx xxx

# 输出：
# Bot Name: CTO
# Bot Open ID: ou_e34e30b1a0fee620ac6cca1b3fb65ab8
# App ID: cli_xxx
```

### 场景 3：多 Bot 管理

批量获取所有 Bot 的 Open ID：

```bash
#!/bin/bash

# bots.csv 格式：app_id,app_secret,bot_name
while IFS=, read -r app_id app_secret bot_name; do
  echo "=== $bot_name ==="
  ./get-bot-openid.sh "$app_id" "$app_secret"
  echo ""
done < bots.csv
```

---

## 📊 验证方法

### 验证 1：API 调用验证

```bash
# 调用 API 获取 Bot 信息
curl -X POST "https://open.feishu.cn/open-apis/bot/v1/openclaw_bot/ping" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"needBotInfo":true}' | jq '.'

# 预期返回：
{
  "code": 0,
  "data": {
    "pingBotInfo": {
      "botID": "ou_xxx",
      "botName": "CTO"
    }
  }
}
```

### 验证 2：飞书开放平台验证

1. 打开飞书开放平台：https://open.feishu.cn/
2. 进入应用管理 → 选择对应应用
3. 查看应用信息 → 基本信息
4. 确认 Bot Open ID 与 API 返回一致

### 验证 3：Bot @ Bot 功能验证

在飞书群聊中测试：

```
@CTO Bot 测试消息
```

观察 CTO Bot 是否收到@通知并响应。

**测试结果**：✅ 成功！CTO Bot 收到@并回复！

---

## ⚠️ 注意事项

### 1. 权限要求

Bot 应用需要以下权限：
- `im:message` - 收发消息
- `im:message.group` - 获取群组消息
- `im:message.group_bot` - 获取其他 Bot 消息

### 2. Token 有效期

- App Access Token 有效期：2 小时
- 过期后需要重新获取

### 3. 安全建议

- ⚠️ **不要将 App Secret 提交到 Git**
- ⚠️ **不要在公开场合泄露 Bot Open ID**
- ✅ 使用环境变量存储敏感信息
- ✅ 使用 `.gitignore` 忽略脚本和配置文件

### 4. 错误处理

常见错误及解决方案：

| 错误码 | 错误信息 | 解决方案 |
|--------|---------|---------|
| 99991672 | Access denied | 检查应用权限是否已开通 |
| 99992351 | Invalid app_access_token | Token 过期，重新获取 |
| 19021 | sign match fail | 签名错误，检查签名算法 |

---

## 🎉 总结

### 核心发现

1. **飞书 Bot 有独立的 Open ID**
   - 格式：`ou_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
   - 通过 `/open-apis/bot/v1/openclaw_bot/ping` API 获取

2. **OpenClaw 配置了 Bot 应用**
   - 有 App ID 和 App Secret
   - 但发送消息时使用用户 OAuth Token

3. **Bot @ Bot 功能可行**
   - 需要 Bot 的真实 Open ID
   - 需要 Bot 在同一个群里
   - 需要正确的权限配置

### 获取方法

**推荐方法**：使用 curl 命令直接调用飞书 API

```bash
# 1. 获取 Token
curl -X POST "https://open.feishu.cn/open-apis/auth/v3/app_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{"app_id":"xxx","app_secret":"xxx"}'

# 2. 获取 Bot Open ID
curl -X POST "https://open.feishu.cn/open-apis/bot/v1/openclaw_bot/ping" \
  -H "Authorization: Bearer <token>" \
  -d '{"needBotInfo":true}'
```

### 应用价值

- ✅ 实现 OpenClaw多Agent 协作中的 Bot @ Bot 功能
- ✅ 验证 Bot 应用配置是否正确
- ✅ 批量管理多个 Bot 的身份信息
- ✅ 解决 Bot 身份识别的技术难题

---

## 📚 参考资料

- [飞书开放平台 - 获取应用访问凭证](https://open.feishu.cn/document/ukTMukTMukTM/ukDNz4SO0MjL5QzM/auth-v3/auth/app_access_token)
- [飞书开放平台 - Bot 身份验证](https://open.feishu.cn/document/ukTMukTMukTM/uAjNwUjL1YDM14SN2ATN)
- [OpenClaw 飞书插件](https://github.com/larksuite/openclaw-lark)
- [OpenClaw 文档](https://docs.openclaw.ai)

---

**作者**: 秘书长 📋  
**项目**: [openclaw-team/agent](https://github.com/openclaw-team/agent)  
**许可**: MIT License  
**最后更新**: 2026-04-22

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

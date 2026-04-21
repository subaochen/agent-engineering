# Clash Verge Ubuntu 部署与自动节点调度指南

> **研究时间**: 2026-04-21  
> **适用系统**: Ubuntu 20.04/22.04/24.04  
> **客户端**: Clash Verge Rev (基于 Tauri 的现代化 GUI 客户端)

---

## 📋 目录

1. [Clash Verge 简介](#clash-verge-简介)
2. [Ubuntu 安装部署](#ubuntu-安装部署)
3. [配置订阅与节点](#配置订阅与节点)
4. [自动节点调度](#自动节点调度)
5. [高级配置](#高级配置)
6. [故障排查](#故障排查)

---

## 🎯 Clash Verge 简介

### 什么是 Clash Verge Rev？

**Clash Verge Rev** 是基于 Tauri 框架的现代化 Clash GUI 客户端，支持 Windows、macOS 和 Linux。

**核心特性**：
- ✅ 基于 Rust + Tauri 2，性能强劲
- ✅ 内置 Clash.Meta (mihomo) 内核
- ✅ 简洁美观的 UI，支持自定义主题
- ✅ 系统代理和 TUN 模式
- ✅ 可视化节点和规则编辑
- ✅ WebDav 配置备份

**GitHub**: https://github.com/clash-verge-rev/clash-verge-rev

---

## 🚀 Ubuntu 安装部署

### 方式 A：DEB 包安装（推荐）

#### Step 1: 下载最新版本

```bash
# 创建下载目录
mkdir -p ~/Downloads/clash-verge
cd ~/Downloads/clash-verge

# 下载最新 DEB 包（替换为最新版本号）
wget https://github.com/clash-verge-rev/clash-verge-rev/releases/download/v1.7.8/Clash.Verge_1.7.8_linux_x86_64.deb

# 或下载 ARM64 版本（树莓派等）
# wget https://github.com/clash-verge-rev/clash-verge-rev/releases/download/v1.7.8/Clash.Verge_1.7.8_linux_aarch64.deb
```

#### Step 2: 安装依赖

```bash
# 更新软件源
sudo apt update

# 安装必要依赖
sudo apt install -y libgtk-3-0 libwebkit2gtk-4.0-37 libappindicator3-1 libjavascriptcoregtk-4.0-37 libayatana-appindicator3-1 librsvg2-common
```

#### Step 3: 安装 DEB 包

```bash
# 安装
sudo dpkg -i Clash.Verge_*.deb

# 如果有依赖问题，执行
sudo apt-get install -f -y
```

#### Step 4: 启动应用

```bash
# 方式 1：从应用程序菜单启动
# 搜索 "Clash Verge"

# 方式 2：命令行启动
clash-verge

# 方式 3：后台启动
nohup clash-verge > /dev/null 2>&1 &
```

---

### 方式 B：AppImage（免安装）

#### Step 1: 下载 AppImage

```bash
cd ~/Apps
wget https://github.com/clash-verge-rev/clash-verge-rev/releases/download/v1.7.8/Clash.Verge_1.7.8_linux_x86_64.AppImage
```

#### Step 2: 赋予执行权限

```bash
chmod +x Clash.Verge_*.AppImage
```

#### Step 3: 运行

```bash
./Clash.Verge_*.AppImage
```

#### Step 4: （可选）集成到系统菜单

```bash
# 创建桌面文件
cat > ~/.local/share/applications/clash-verge.desktop << EOF
[Desktop Entry]
Name=Clash Verge
Exec=$HOME/Apps/Clash.Verge_*.AppImage
Icon=clash-verge
Type=Application
Categories=Network;Utility;
EOF
```

---

### 方式 C：AUR（Arch Linux）

```bash
# 使用 yay
yay -S clash-verge-rev

# 或使用 paru
paru -S clash-verge-rev
```

---

## ⚙️ 配置订阅与节点

### Step 1: 首次启动

1. 启动 Clash Verge
2. 首次运行会提示选择语言（支持简体中文）
3. 同意用户协议

### Step 2: 导入订阅

**方式 A：HTTP 链接导入**

1. 点击 **"配置文件"**
2. 点击 **"+"** 添加
3. 选择 **"HTTP"**
4. 输入订阅链接
5. 点击 **"确定"**

**方式 B：本地文件导入**

1. 准备 Clash 配置文件（YAML 格式）
2. 点击 **"配置文件"** → **"+"** → **"本地"**
3. 选择 YAML 文件
4. 点击 **"确定"**

### Step 3: 选择节点

1. 点击 **"代理"** 标签
2. 选择代理组（如"自动选择"、"手动选择"）
3. 点击节点进行切换

### Step 4: 开启系统代理

1. 点击底部 **"系统代理"** 开关
2. 或使用快捷键 `Ctrl + Shift + P`

---

## 🔄 自动节点调度

### 方案 A：内置自动选择（推荐）⭐⭐⭐⭐⭐

**Clash Verge 内置自动选择功能**：

#### 配置方法

1. **打开配置文件**
   ```yaml
   # 在订阅配置中添加
   proxy-groups:
     - name: 🚀 自动选择
       type: url-test
       proxies:
         - .*
       url: https://www.google.com/favicon.ico
       interval: 300
       tolerance: 50
   ```

2. **参数说明**
   ```yaml
   type: url-test          # 自动测试类型
   interval: 300           # 每 300 秒测试一次
   tolerance: 50           # 容差 50ms，超过才切换
   url: https://google.com # 测试网址
   ```

3. **效果**
   - ✅ 自动选择延迟最低的节点
   - ✅ 定期测试节点质量
   - ✅ 节点故障自动切换

---

### 方案 B：脚本自动调度 ⭐⭐⭐⭐

**使用脚本定期测试并切换节点**：

#### Step 1: 创建测试脚本

```bash
cat > ~/scripts/clash-speedtest.sh << 'EOF'
#!/bin/bash
# Clash 节点速度测试与自动切换

API_URL="http://127.0.0.1:9090"
PROXIES_API="$API_URL/proxies"
CONFIG_API="$API_URL/configs"

# 获取所有节点
get_proxies() {
    curl -s "$PROXIES_API" | jq -r '.proxies["🚀 自动选择"].all[]'
}

# 测试节点延迟
test_latency() {
    local proxy=$1
    curl -s -w "%{time_total}" \
         -o /dev/null \
         -x "http://127.0.0.1:7890" \
         --connect-timeout 5 \
         https://www.google.com/favicon.ico 2>/dev/null
}

# 选择最快节点
find_fastest() {
    local fastest=""
    local min_latency=999999
    
    for proxy in $(get_proxies); do
        latency=$(test_latency "$proxy")
        if [ -n "$latency" ] && (( $(echo "$latency < $min_latency" | bc -l) )); then
            min_latency=$latency
            fastest=$proxy
        fi
        echo "测试 $proxy: ${latency}s"
    done
    
    echo "最快节点：$fastest (${min_latency}s)"
    echo "$fastest"
}

# 切换节点
switch_proxy() {
    local proxy=$1
    curl -X PUT "$PROXIES_API/🚀 自动选择" \
         -H "Content-Type: application/json" \
         -d "{\"name\":\"$proxy\"}"
}

# 主流程
main() {
    echo "开始测试节点..."
    fastest=$(find_fastest)
    
    if [ -n "$fastest" ]; then
        echo "切换到最快节点：$fastest"
        switch_proxy "$fastest"
    else
        echo "未找到可用节点"
    fi
}

main
EOF

chmod +x ~/scripts/clash-speedtest.sh
```

#### Step 2: 创建定时任务

```bash
# 编辑 crontab
crontab -e

# 添加：每 5 分钟测试并切换最快节点
*/5 * * * * /home/youruser/scripts/clash-speedtest.sh >> /tmp/clash-speedtest.log 2>&1
```

---

### 方案 C：使用 Mihomo 内置功能 ⭐⭐⭐⭐⭐

**Mihomo (Clash.Meta) 内核自带自动调度**：

#### 配置示例

```yaml
# config.yaml
proxy-groups:
  - name: 🚀 自动选择
    type: url-test
    proxies:
      - HK 香港
      - US 美国
      - JP 日本
      - SG 新加坡
    url: https://www.google.com/favicon.ico
    interval: 300
    tolerance: 50
    
  - name: 🎬 流媒体
    type: select
    proxies:
      - 🚀 自动选择
      - HK 香港
      - US 美国
      
  - name: 🤖 AI 服务
    type: url-test
    proxies:
      - US 美国
      - SG 新加坡
    url: https://api.openai.com
    interval: 180
    tolerance: 30

rules:
  - DOMAIN-SUFFIX,google.com,🚀 自动选择
  - DOMAIN-SUFFIX,openai.com,🤖 AI 服务
  - DOMAIN-SUFFIX,netflix.com,🎬 流媒体
  - MATCH,🚀 自动选择
```

#### 参数说明

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `type` | 测试类型 | `url-test` |
| `interval` | 测试间隔（秒） | `300` |
| `tolerance` | 容差（毫秒） | `50` |
| `url` | 测试网址 | Google favicon |

---

## 🎯 高级配置

### TUN 模式（全局代理）

**启用 TUN 模式**：

```yaml
# Clash Verge 设置
tun:
  enable: true
  stack: system
  dns-hijack:
    - any:53
  auto-route: true
  auto-detect-interface: true
```

**效果**：
- ✅ 所有流量走代理（包括终端）
- ✅ 无需手动配置环境变量
- ✅ 类似 VPN 体验

---

### 环境变量代理

**临时设置**：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=http://127.0.0.1:7890
```

**永久设置**：

```bash
cat >> ~/.bashrc << 'EOF'

# Clash Verge 代理
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
EOF

source ~/.bashrc
```

---

### API 控制

**Clash Verge API**：

```bash
# API 地址
API_URL="http://127.0.0.1:9090"

# 查看当前配置
curl "$API_URL/configs"

# 查看代理列表
curl "$API_URL/proxies"

# 切换节点
curl -X PUT "$API_URL/proxies/🚀 自动选择" \
     -H "Content-Type: application/json" \
     -d '{"name":"HK 香港"}'

# 查看日志
curl "$API_URL/logs"
```

---

## 🔧 故障排查

### 问题 1: 无法启动

```bash
# 查看错误日志
clash-verge 2>&1 | tee /tmp/clash-verge-error.log

# 检查依赖
ldd /usr/bin/clash-verge | grep "not found"

# 重新安装依赖
sudo apt-get install -f -y
```

### 问题 2: 无法连接网络

```bash
# 检查系统代理
echo $http_proxy
echo $https_proxy

# 检查 Clash 端口
netstat -tlnp | grep clash

# 重启 Clash Verge
pkill clash-verge
clash-verge
```

### 问题 3: 节点不自动切换

```bash
# 检查配置文件
cat ~/.config/clash-verge/prf/*.yaml | grep -A 10 "url-test"

# 手动测试节点
~/scripts/clash-speedtest.sh

# 查看 Clash 日志
tail -f ~/.config/clash-verge/logs/*.log
```

### 问题 4: TUN 模式不工作

```bash
# 检查权限
sudo setcap cap_net_bind_service=+ep /usr/bin/clash-verge

# 检查内核模块
lsmod | grep tun

# 加载 TUN 模块
sudo modprobe tun
```

---

## 📊 性能优化

### 1. 选择合适内核

**Clash Verge 支持切换内核**：

```
设置 → 内核设置 → 选择 Clash.Meta (推荐)
```

**优势**：
- ✅ 支持更多协议
- ✅ 性能更好
- ✅ 自动调度更智能

### 2. 优化测试间隔

```yaml
# 频繁测试（网络不稳定）
interval: 60

# 正常测试（推荐）
interval: 300

# 节省流量
interval: 600
```

### 3. 使用 QUIC 协议

**如果机场支持 QUIC**：

```yaml
# 配置中添加
experimental:
  quic-go-disable-gso: false
```

**优势**：
- ✅ 更低延迟
- ✅ 抗丢包能力强
- ✅ 晚高峰更稳定

---

## 🎁 最佳实践

### 日常使用配置

```yaml
# 推荐配置
proxy-groups:
  # 自动选择（默认）
  - name: 🚀 自动选择
    type: url-test
    proxies: [.*]
    interval: 300
    tolerance: 50
    
  # 流媒体专用
  - name: 🎬 流媒体
    type: url-test
    proxies: [🚀 自动选择]
    interval: 180
    
  # AI 服务专用
  - name: 🤖 AI 服务
    type: url-test
    proxies: [US.*, SG.*]
    interval: 180
    tolerance: 30

rules:
  # AI 服务
  - DOMAIN-SUFFIX,openai.com,🤖 AI 服务
  - DOMAIN-SUFFIX,anthropic.com,🤖 AI 服务
  
  # 流媒体
  - DOMAIN-SUFFIX,netflix.com,🎬 流媒体
  - DOMAIN-SUFFIX,spotify.com,🎬 流媒体
  
  # 默认
  - MATCH,🚀 自动选择
```

### 备份配置

```bash
# 备份 Clash Verge 配置
tar -czf clash-verge-backup-$(date +%Y%m%d).tar.gz \
    ~/.config/clash-verge/

# 恢复到其他机器
tar -xzf clash-verge-backup-*.tar.gz -C ~/
```

---

## 📚 参考资源

### 官方文档
- GitHub: https://github.com/clash-verge-rev/clash-verge-rev
- 发布页面：https://github.com/clash-verge-rev/clash-verge-rev/releases
- 文档：https://clash-verge-rev.github.io/

### 相关项目
- Mihomo (Clash.Meta): https://github.com/MetaCubeX/mihomo
- Tauri: https://github.com/tauri-apps/tauri

### 社区资源
- Telegram 频道
- Discord 社区
- Reddit r/Clash

---

*最后更新：2026-04-21*  
*维护者：秘书长 AI 团队*

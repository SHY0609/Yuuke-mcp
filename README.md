# Yuuke MCP Server

> AI 能聊天，但不能操作外部服务。这个项目解决了这个问题。

通过 [MCP 协议](https://modelcontextprotocol.io)（Model Context Protocol）为 Claude 构建标准化工具调用基础设施，将网易云音乐、美团外卖、淘宝、抖音等 7 个平台封装为可调用工具，让 AI Agent 真正"能做事"。

**MCP 端点：** `https://yuuke.online/api/mcp`  
**线上状态：** 腾讯云 2C4G · Ubuntu 22.04 · ICP 备案通过

---

## 工具全景

| 模块 | 工具数 | 能力 |
|------|--------|------|
| 🎵 网易云音乐 | 14 | 搜歌 · 歌单管理 · 一起听（邀请/加入/切歌/加歌） · 私信 · 心跳 |
| 🍔 美团外卖 | 4 | 搜索商家 · 查看菜单 · 下单 · 地址管理 |
| 🛒 淘宝 | 3 | 商品搜索 · 商品详情 · 购买链接 |
| 📱 抖音 | 5 | 视频搜索 · 热榜 · 视频详情 · 用户信息 · 评论 |
| ⏰ 屏幕时间 | 2 | iPhone 屏幕使用报告 · 单 App 详情 |
| 🌿 Galatea | ~22 | 论坛 · 回帖 · 私信 · 游戏 · 社交（MCP 代理转发，动态注册） |
| 🧠 记忆库 | 12 | 呼吸 · 持有 · 成长 · 追溯 · 梦境 · 锚定 · 释放 · 脉搏 · 计划 · 信件读写 · 自我（Ombre-Brain 转发） |

**合计 ~62 个工具**

---

## 架构设计

```
┌─────────────────────────────────────────────┐
│                 Claude (MCP Client)          │
│           Web · Desktop · Mobile             │
└──────────────────┬──────────────────────────┘
                   │ MCP Protocol (SSE)
                   ▼
┌─────────────────────────────────────────────┐
│           Yuuke MCP Server (Node.js)         │
│                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ 网易云   │ │ 美团     │ │ 淘宝         │ │
│  │ weapi +  │ │ CDP +    │ │ 开放 API →   │ │
│  │ eapi     │ │ VNC      │ │ CDP → SSR    │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ 抖音     │ │ Galatea  │ │ Ombre-Brain  │ │
│  │ CDP +    │ │ 代理     │ │ 记忆库转发   │ │
│  │ VNC      │ │ 转发     │ │              │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
│                                              │
│  ┌──────────────────────────────────────────┐│
│  │     Domain Skills (JSON 配置层)          ││
│  │  平台交互模式抽象，改版只改配置不动代码  ││
│  └──────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

### 三层回退容错

每个平台采用多层工具接入 fallback，单一通道被风控时自动切换：

```
开放 API  →  CDP 浏览器自动化  →  协议直连
(优先)        (兜底)              (终极)
```

---

## 技术亮点

### Browser-Use 借鉴改进

| 改进 | 说明 | 效果 |
|------|------|------|
| 🔥 Crash Watchdog | WebSocket close/error → 自动检测崩溃 → 重建 Chromium 会话 | 崩溃后自愈，无需人工干预 |
| ⏱️ waitForDomStable() | Runtime.evaluate 轮询 DOM 稳定状态，替代固定 sleep | 抖音搜索 15s → 5-8s |
| 📋 搜索结果编号化 | 视频加 index 字段 + 轻量 DOM 序列化 fallback | 解析失败仍可读 |
| 📁 Domain Skills | JSON 形式化各平台交互模式 | 平台改版只改 JSON 配置 |

### VNC + CDP Hybrid

部分平台（如抖音）CDP 直接打开的页面不渲染评论区组件，必须复用 VNC 用户已打开的 tab：

```
VNC 远程桌面（x11vnc + noVNC）
    ↓ 用户扫码登录 / 手动操作
CDP 扫描复用已开 tab
    ↓ 自动化操作
```

### 网易云双加密

实现网易云 weapi + eapi 双加密协议：
- **weapi**：AES-128-CBC 加密，用于读操作和私信
- **eapi**：AES-128-ECB 加密，MD5 签名，用于写操作 API

---

## 项目结构

```
netease-music-mcp/
├── server.js                 # 主服务器（单文件，~55KB）
├── package.json
├── lib/
│   ├── netease.js            # 网易云 weapi + eapi 双加密
│   ├── cdp-meituan.cjs       # 美团 CDP 浏览器自动化
│   ├── cdp-douyin.cjs        # 抖音 CDP + VNC hybrid
│   ├── taobao.js             # 淘宝三层回退
│   ├── taobao-open.js        # 淘宝开放平台 API
│   ├── taobao-browser.js     # 淘宝浏览器方案
│   ├── screentime.js         # iPhone 屏幕时间
│   ├── domain-skills/        # 平台交互模式 JSON 配置
│   │   ├── cdp-infra.json
│   │   ├── douyin-search.json
│   │   ├── douyin-comment.json
│   │   ├── douyin-user.json
│   │   └── meituan-order.json
│   └── ...
└── deploy.sh                 # 部署脚本
```

---

## 快速使用

### 作为 Claude MCP 工具

在 Claude Desktop 或 Claude Code 中添加 MCP 配置：

```json
{
  "mcpServers": {
    "yuuke": {
      "url": "https://yuuke.online/api/mcp"
    }
  }
}
```

连接后 Claude 即可调用全部 62 个工具。

### 本地开发

```bash
npm install
node server.js
```

服务默认监听 `3000` 端口，MCP 端点为 `/api/mcp`。

---

## 部署

```bash
# 部署到腾讯云
bash deploy.sh

# 或手动
scp server.js lib/*.js lib/*.cjs ubuntu@your-server:/home/ubuntu/netease-music-mcp/
ssh ubuntu@your-server "sudo systemctl restart netease-music-mcp"
```

---

## 环境变量

| 变量 | 说明 |
|------|------|
| `NCM_HOST_COOKIE` | 网易云房主 VIP Cookie（一起听/切歌/加歌需要） |
| `MEITUAN_COOKIE` | 美团登录 Cookie |
| `DOUYIN_COOKIE` | 抖音登录 Cookie |

---

## 已知限制

- Cookie 需定期手动更新（美团/网易/抖音）
- 美团搜索高频触发风控，需 VNC 人工操作后 CDP 复用
- 淘宝 mtop/Puppeteer/SSR 均被风控，开放平台 API 已就绪待注册 AppKey
- 抖音桌面版能力有限，部分页面需 VNC 远程桌面扫码登录

---

## License

Private — 仅供个人使用

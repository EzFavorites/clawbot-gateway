# ClawBot Gateway 用户指南

> 微信多后端消息网关 — 一个微信账号，连接多个 AI 后端

---

## 目录

1. [概述](#1-概述)
2. [快速开始](#2-快速开始)
3. [安装与部署](#3-安装与部署)
4. [配置说明](#4-配置说明)
5. [Web 管理界面](#5-web-管理界面)
6. [微信账号管理](#6-微信账号管理)
7. [后端适配器](#7-后端适配器)
8. [路由规则](#8-路由规则)
9. [消息命令](#9-消息命令)
10. [通知 API](#10-通知-api)
11. [API 参考](#11-api-参考)
12. [安全](#12-安全)
13. [维护与运维](#13-维护与运维)
14. [常见问题](#14-常见问题)

---

## 1. 概述

### 1.1 什么是 ClawBot Gateway

ClawBot Gateway 是一个微信消息网关，它通过腾讯 iLink 协议接入微信，支持同时连接多个 AI 后端（如 Claude、DeepSeek、GPT、Hermes 等），用户可以通过消息命令在多个后端之间切换。

### 1.2 核心架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ClawBot Gateway                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─── iLink 客户端（Connector）─────────────────────────────────┐  │
│  │  连接腾讯 iLink API，接收/发送微信消息                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─── iLink 服务端（透明代理）──────────────────────────────────┐  │
│  │  对外提供 iLink 兼容 API，供虚拟 Bot 连接                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─── 适配器层 ─────────────────────────────────────────────────┐  │
│  │  ilink_proxy      外部 iLink 服务（生成虚拟 Bot 配置）        │  │
│  │  openai_compatible OpenAI 兼容 API（Claude/DeepSeek等）       │  │
│  │  echo             调试回显                                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─── Notify 通知模块 ─────────────────────────────────────────┐  │
│  │  允许外部系统向微信用户发送消息                                │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌─── Pipeline 消息管道 ───────────────────────────────────────┐  │
│  │  命令解析 → 路由匹配 → 后端处理 → 回复构建                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.3 核心概念

| 概念 | 说明 |
|------|------|
| **iLink 客户端** | 连接腾讯 iLink 服务的组件，负责收发微信消息、多账号管理和 QR 扫码登录 |
| **iLink 服务端** | 对外提供 iLink 兼容 API，供虚拟 Bot 连接，实现透明代理 |
| **虚拟 Bot** | 基于一个真实微信账号虚拟出的独立 Bot 实例，每个虚拟 Bot 有独立的 token |
| **后端适配器** | 连接 AI 服务的适配器，支持 OpenAI 兼容 API、iLink 代理、调试回显等 |
| **路由规则** | 定义消息如何匹配和路由到指定后端的规则引擎 |
| **Pipeline** | 消息处理管道，统一处理命令解析、路由匹配和后端调用 |
| **Notify 通知** | 允许外部系统通过 API 向微信用户发送消息的独立通道 |

---

## 2. 快速开始

### 2.1 一分钟启动

```bash
# 1. 克隆仓库
git clone https://github.com/EzFavorites/clawbot-gateway.git
cd clawbot-gateway

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env，设置 CLAWBOT_LOGIN_PASSWORD

# 3. 启动
docker compose up -d

# 4. 查看日志（获取自动生成的登录密码）
docker compose logs -f

# 5. 打开管理面板
# 浏览器访问 http://localhost:6798
```

### 2.2 基本使用流程

```
1. 启动 Gateway → 2. 登录管理面板 → 3. 扫码登录微信 → 4. 添加后端 → 5. 配置路由 → 6. 开始使用
```

---

## 3. 安装与部署

### 3.1 Docker 部署（推荐）

```bash
# 拉取镜像
docker pull ghcr.io/ezfavorites/clawbot-gateway:latest

# 创建数据目录
mkdir -p data

# 启动
docker run -d \
  --name clawbot-gateway \
  --restart unless-stopped \
  -p 6798:6798 \
  -v $(pwd)/data:/app/data \
  -e CLAWBOT_LOGIN_PASSWORD=your_password \
  ghcr.io/ezfavorites/clawbot-gateway:latest
```

### 3.2 docker-compose 部署

```yaml
services:
  clawbot-gateway:
    image: ghcr.io/ezfavorites/clawbot-gateway:latest
    container_name: clawbot-gateway
    restart: unless-stopped
    ports:
      - "6798:6798"
    environment:
      - TZ=Asia/Shanghai
      - CLAWBOT_DB_PATH=data/clawbot.db
      - CLAWBOT_HOST=0.0.0.0
      - CLAWBOT_PORT=6798
      - CLAWBOT_LOG_LEVEL=info
      # CLAWBOT_LOGIN_PASSWORD=your_password
    volumes:
      - ./data:/app/data
```

### 3.3 原生部署

```bash
# 需要 Go 1.22+ 和 Node.js 22+

# 构建后端
go build -o clawbot-gateway .

# 构建前端
cd web && npm ci && npm run build && cd ..

# 运行
./clawbot-gateway
```

### 3.4 桌面端（Tauri）

macOS（Apple Silicon）和 Windows 桌面客户端可从 [Releases](https://github.com/EzFavorites/clawbot-gateway/releases) 页面下载 DMG / MSI / EXE 安装包。

---

## 4. 配置说明

### 4.1 环境变量（启动配置）

| 变量 | 默认值 | 必填 | 说明 |
|------|--------|------|------|
| `CLAWBOT_DB_PATH` | `data/clawbot.db` | 否 | SQLite 数据库文件路径 |
| `CLAWBOT_HOST` | `0.0.0.0` | 否 | 监听地址 |
| `CLAWBOT_PORT` | `6798` | 否 | 监听端口 |
| `CLAWBOT_LOGIN_PASSWORD` | 自动生成 | 否 | 管理面板登录密码（不设置则自动生成并打印到日志） |
| `CLAWBOT_JWT_SECRET` | 自动生成 | 否 | JWT 签名密钥 |
| `CLAWBOT_LOG_LEVEL` | `info` | 否 | 日志级别：`debug`, `info`, `warn`, `error` |
| `WEIXIN_TOKEN` | — | 否 | 微信 Bot Token（扫码登录后自动获取） |
| `WEIXIN_ACCOUNT_ID` | — | 否 | 微信账号 ID（扫码登录后自动获取） |
| `WEIXIN_BASE_URL` | `https://ilinkai.weixin.qq.com` | 否 | 腾讯 iLink API 地址 |
| `WEIXIN_POLL_TIMEOUT` | `35` | 否 | 轮询超时（秒） |
| `WEIXIN_BOT_TYPE` | `3` | 否 | Bot 类型 |

### 4.2 运行时配置（数据库）

以下配置通过 Web 管理界面设置，不需要重启服务：

| 配置项 | 说明 |
|--------|------|
| 后端适配器 | 添加/编辑/删除 AI 后端和 iLink 代理 |
| 路由规则 | 配置消息路由条件和目标后端 |
| 系统设置 | JWT 有效期、会话超时等 |
| 通知 Token | 管理外部系统通知通道的访问令牌 |
| 登录密码 | 修改管理面板登录密码 |

---

## 5. Web 管理界面

### 5.1 登录

首次启动时，密码会自动生成并打印到日志：

```bash
docker compose logs -f
# 输出: [INFO] 登录密码: 3f7a2b9c...
```

使用该密码登录 `http://localhost:6798`。

### 5.2 功能页

| 页面 | 功能 |
|------|------|
| **仪表盘** | 系统概览、连接状态、消息统计 |
| **账号管理** | 微信账号管理、QR 扫码登录、查看虚拟 Bot 配置 |
| **后端管理** | 添加/编辑/删除后端适配器 |
| **路由管理** | 配置消息路由规则和条件 |
| **通知管理** | 管理通知 Token（创建、删除） |
| **系统设置** | 修改密码、JWT 配置、会话策略 |
| **日志查看** | 实时日志流（SSE 推送） |

### 5.3 修改密码

在设置页面可以修改登录密码，需要提供旧密码验证。

---

## 6. 微信账号管理

### 6.1 添加微信账号

#### 方法一：扫码登录

1. 进入管理面板 → 账号管理
2. 点击"添加账号" → 选择"扫码登录"
3. 页面显示二维码，使用微信扫码
4. 扫码成功后，账号自动添加到列表

#### 方法二：手动配置

在 `.env` 文件中配置 `WEIXIN_TOKEN` 和 `WEIXIN_ACCOUNT_ID`。

### 6.2 多账号管理

Gateway 支持同时管理多个微信账号。每个账号独立连接 iLink 服务，互不干扰。

### 6.3 虚拟 Bot

当配置 `ilink_proxy` 类型后端适配器时，Gateway 会自动创建虚拟 Bot。虚拟 Bot 是独立的 iLink 客户端实例，有独立的连接配置（account_id、user_id、base_url、token）。

**虚拟 Bot 的特点：**
- 基于真实微信账号虚拟化，不需要额外扫码
- 每个虚拟 Bot 有独立的连接 token
- 外部服务通过虚拟 Bot 的 iLink API 收发消息
- 虚拟 Bot 的 token 仅存储在数据库中，不暴露真实微信凭证

### 6.4 查看虚拟 Bot 配置

在管理面板 → 连接管理页面，可以查看所有连接适配器的：
- 虚拟 Bot ID（`account_id`）
- 用户 ID（`user_id`）
- 基础 URL（`base_url`）
- 连接 token
- 连接状态

---

## 7. 后端适配器

### 7.1 适配器类型

| 类型 | 类别 | 说明 | 创建虚拟 Bot |
|------|------|------|-------------|
| `ilink_proxy` | 连接适配器 | 代理到外部 iLink 服务（如 Hermes、OpenClaw） | 是 |
| `openai_compatible` | 后端适配器 | 兼容 OpenAI API 格式的服务（Claude、DeepSeek、GPT 等） | 否 |
| `echo` | 后端适配器 | 调试回显，返回收到的消息 | 否 |

### 7.2 添加后端适配器

进入管理面板 → 后端管理 → 添加后端。

#### OpenAI 兼容后端

```json
{
  "type": "openai_compatible",
  "name": "DeepSeek",
  "config": {
    "api_url": "https://api.deepseek.com/v1/chat/completions",
    "api_key": "sk-xxxx",
    "model": "deepseek-chat",
    "max_tokens": 4096
  }
}
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `api_url` | 是 | OpenAI 兼容 API 地址 |
| `api_key` | 是 | API 密钥 |
| `model` | 是 | 模型名称 |
| `max_tokens` | 否 | 最大输出 token 数（默认 4096） |
| `temperature` | 否 | 温度参数（默认 0.7） |
| `system_prompt` | 否 | 自定义系统提示词 |

#### iLink 代理后端

```json
{
  "type": "ilink_proxy",
  "name": "Hermes",
  "config": {
    "base_url": "https://ilinkai.weixin.qq.com",
    "token": "external_bot_token",
    "account_id": "external_account_id"
  }
}
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `base_url` | 是 | 外部 iLink API 地址 |
| `token` | 是 | 外部 Bot Token |
| `account_id` | 否 | 外部账号 ID |

#### 回显后端

```json
{
  "type": "echo",
  "name": "回显调试"
}
```

无需额外配置，收到消息后原样返回。

### 7.3 适配器自注册

新增适配器类型时，在 `internal/adapter/` 目录下创建新文件，实现 `BackendAdapter` 或 `ConnectionAdapter` 接口，在 `init()` 中调用 `RegisterAdapter()` 注册。

---

## 8. 路由规则

### 8.1 路由逻辑

消息路由按照以下优先级判断：

```
1. 用户会话级覆写（/use 命令设置）→ 2. 路由规则匹配 → 3. 默认后端
```

### 8.2 路由条件

每条路由规则可以包含多个条件，支持逻辑组合：

| 条件类型 | 匹配字段 | 匹配方式 | 说明 |
|----------|---------|---------|------|
| `keyword` | 消息内容 | 关键词匹配 | 消息包含指定关键词 |
| `regexp` | 消息内容 | 正则匹配 | 消息匹配正则表达式 |
| `prefix` | 消息内容 | 前缀匹配 | 消息以指定前缀开头 |
| `user` | 发送者 | 精确匹配 | 指定用户的消息 |
| `group` | 群组 | 精确匹配 | 指定群组的消息 |
| `msg_type` | 消息类型 | 精确匹配 | 指定消息类型（text/image/file） |

### 8.3 条件逻辑

```
单条件规则：        条件匹配 → 路由到目标后端
多条件规则（AND）：  所有条件都匹配 → 路由到目标后端
多条件规则（OR）：   任一条件匹配 → 路由到目标后端
NOT 条件：          条件不匹配 → 路由到目标后端
```

### 8.4 规则优先级

路由规则按优先级排序，优先级高的规则先匹配。匹配成功后不再继续匹配后续规则。

### 8.5 消息流程

```
收到消息
  ↓
命令解析 ── 匹配 /command → 执行命令（/use、/help、/backends 等）
  ↓
路由匹配 ── 匹配规则 → 路由到指定后端
  ↓
后端处理 ── 调用后端 → 获取回复
  ↓
回复构建 ── 构建消息 → 发送回复
```

---

## 9. 消息命令

用户可以通过微信消息与 Gateway 交互，使用 `/` 前缀的命令。

### 9.1 基础命令

| 命令 | 说明 |
|------|------|
| `/help` | 显示帮助信息 |
| `/use` | 显示当前后端状态 |
| `/use <backend>` | 切换到指定后端（持久化，后续消息都走该后端） |
| `/use main` | 回到主命令模式（清除后端选择） |
| `/backends` | 列出所有可用后端 |
| `/<backend>` | 一次性转发到该后端（不切换，仅本次消息） |

### 9.2 命令示例

```
用户：/use hermes
Gateway：已切换到 Hermes，后续消息将发送到 Hermes

用户：你好
（消息自动转发到 Hermes）

用户：/use main
Gateway：已切换到主命令模式

用户：/hermes 帮我写个 Python 脚本
（本次消息转发到 Hermes，但不切换）

用户：/backends
Gateway：可用后端：hermes, openclaw, deepseek, echo
```

### 9.3 路由规则命令

当配置了路由规则（如关键词匹配），匹配的消息会自动路由到指定后端，无需手动切换。

**规则命中提示：** 当消息被路由规则匹配时，Gateway 会发送提示消息告知用户消息被路由到了哪个后端。

---

## 10. 通知 API

通知模块允许外部系统向微信用户发送消息。

### 10.1 创建通知 Token

```bash
curl -X POST http://localhost:6798/api/v1/notify/tokens \
  -H "Authorization: Bearer <管理Token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "我的通知通道"}'
```

响应：

```json
{
  "token": "nt_c3RhZ2luZy5kYXRh..."
}
```

> **注意：** Token 创建后仅显示一次，请妥善保存。

### 10.2 发送通知

```bash
curl -X POST http://localhost:6798/api/v1/notify/send \
  -H "Authorization: Bearer nt_c3RhZ2luZy5kYXRh..." \
  -H "Content-Type: application/json" \
  -d '{
    "to_user": "wx_xxxxxxxx",
    "content": "你好，这是一条通知消息"
  }'
```

### 10.3 管理通知 Token

```bash
# 列出所有通知 Token
curl http://localhost:6798/api/v1/notify/tokens \
  -H "Authorization: Bearer <管理Token>"

# 删除通知 Token
curl -X DELETE http://localhost:6798/api/v1/notify/tokens/:id \
  -H "Authorization: Bearer <管理Token>"
```

---

## 11. API 参考

### 11.1 公开端点

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/health` | 健康检查 |
| `POST` | `/auth/login` | 登录（获取 JWT Token） |

### 11.2 iLink API（透明代理）

这些端点对外提供 iLink 兼容 API，供虚拟 Bot 连接。

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/ilink/bot/getupdates` | 长轮询获取消息（从队列消费） |
| `POST` | `/ilink/bot/sendmessage` | 发送消息（透明代理到腾讯 iLink） |
| `POST` | `/ilink/bot/sendtyping` | 发送输入状态 |
| `POST` | `/ilink/bot/getconfig` | 获取虚拟 Bot 配置 |
| `POST` | `/ilink/bot/getuploadurl` | 获取文件上传 URL |

### 11.3 管理 API（需 JWT 鉴权）

#### 后端管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/backends` | 列出所有后端 |
| `POST` | `/api/v1/backends` | 注册后端 |
| `PUT` | `/api/v1/backends/:id` | 更新后端配置 |
| `DELETE` | `/api/v1/backends/:id` | 删除后端 |

#### 路由管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/routes` | 列出所有路由规则 |
| `POST` | `/api/v1/routes` | 添加路由规则 |
| `DELETE` | `/api/v1/routes/:id` | 删除路由规则 |

#### 账号管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/accounts` | 列出所有微信账号 |

#### 连接管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/connections` | 列出连接适配器（含虚拟 Bot token 和连接状态） |
| `GET` | `/api/v1/connections/:id` | 获取连接详情 |
| `GET` | `/api/v1/connections/stats` | 连接统计 |

#### 配置管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/config` | 获取系统配置 |
| `PUT` | `/api/v1/config` | 更新系统配置 |

#### 认证管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `PUT` | `/api/v1/auth/password` | 修改登录密码（需旧密码验证） |
| `GET` | `/api/v1/auth/token` | 获取 API Token |
| `POST` | `/api/v1/auth/token` | 重新生成 API Token |

#### 通知管理

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/notify/tokens` | 列出所有通知 Token |
| `POST` | `/api/v1/notify/tokens` | 创建通知 Token |
| `DELETE` | `/api/v1/notify/tokens/:id` | 删除通知 Token |

### 11.4 日志

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/logs/stream` | SSE 实时日志流（需 Bearer Token 鉴权） |

### 11.5 错误响应

所有 API 端点在出错时返回统一格式：

```json
{
  "error": "错误描述信息"
}
```

| HTTP 状态码 | 说明 |
|-------------|------|
| `200` | 成功 |
| `400` | 请求参数错误 |
| `401` | 未认证或 Token 过期（触发自动登出） |
| `403` | 无权限 |
| `404` | 资源不存在 |
| `500` | 服务器内部错误（不暴露内部细节） |

---

## 12. 安全

### 12.1 认证体系

| 场景 | 认证方式 | 说明 |
|------|---------|------|
| Web UI 登录 | JWT Token | 登录后获取 JWT，用于管理 API 鉴权 |
| API 调用 | Bearer Token | 管理 API 使用 JWT，通知 API 使用通知 Token |
| iLink 服务端 | Bearer Token | 虚拟 Bot 使用独立 token 连接 |
| 通知通道 | Bearer Token | 通知 Token 支持创建和删除 |

### 12.2 安全措施

- **密码安全：** 登录密码通过环境变量设置，未设置时自动生成
- **Token 比较：** 使用 constant-time compare 防止时序攻击
- **JWT 黑名单：** 登出时 JWT 加入黑名单，无法重复使用
- **凭证隔离：** 虚拟 Bot 使用 Gateway 生成的 token，不暴露真实微信凭证
- **请求体限制：** API 请求体大小有限制，防止 DoS 攻击
- **安全响应头：** 自动添加安全相关的 HTTP 响应头

### 12.3 通知 Token 安全

- 通知 Token 创建后仅显示一次
- 支持创建和删除管理
- 独立的 Bearer Token，不与管理 API 混用

---

## 13. 维护与运维

### 13.1 重置密码

```bash
# 生成随机密码
docker exec clawbot-gateway ./clawbot-gateway reset-password

# 指定密码
docker exec clawbot-gateway ./clawbot-gateway reset-password myNewPass
```

### 13.2 查看版本

```bash
docker exec clawbot-gateway ./clawbot-gateway version
# 或
./clawbot-gateway version
```

### 13.3 查看日志

```bash
# Docker 环境
docker logs clawbot-gateway
docker logs -f clawbot-gateway   # 实时跟踪

# 管理面板
# 进入日志页面，可实时查看日志流
```

### 13.4 数据备份

SQLite 数据库文件位于 `data/clawbot.db`，建议定期备份：

```bash
cp data/clawbot.db data/clawbot.db.backup.$(date +%Y%m%d)
```

### 13.5 CLI 子命令

```bash
./clawbot-gateway reset-password [newPassword]
./clawbot-gateway version
./clawbot-gateway help
```

CLI 子命令直接操作数据库，不启动 HTTP 服务。

---

## 14. 常见问题

### 14.1 首次启动后无法登录

```bash
# 查看日志获取自动生成的密码
docker compose logs -f
# 或重置密码
docker exec clawbot-gateway ./clawbot-gateway reset-password
```

### 14.2 微信扫码登录失败

- 确认网络可以访问 `ilinkai.weixin.qq.com`
- 确认二维码未过期（二维码有效期为 5 分钟）
- 尝试重新生成二维码

### 14.3 后端连接失败

- 检查后端配置的 API URL 是否正确
- 确认 API Key 是否有效
- 查看 Gateway 日志获取详细错误信息

### 14.4 消息没有回复

- 检查是否有可用的后端（`/backends`）
- 检查路由规则是否正确配置
- 检查消息是否被命令解析器拦截（以 `/` 开头的消息会被解析为命令）
- 查看日志确认消息是否被正常处理

### 14.5 虚拟 Bot 无法连接

- 确认 iLink 代理后端配置正确
- 检查虚拟 Bot 的 token 是否有效
- 确认 `base_url` 指向正确的 iLink API 地址

### 14.6 如何升级

```bash
# Docker 部署
docker pull ghcr.io/ezfavorites/clawbot-gateway:latest
docker compose down
docker compose up -d

# 数据库会自动兼容，无需迁移
```

### 14.7 端口冲突

修改 `.env` 中的 `CLAWBOT_PORT`，或在 `docker-compose.yml` 中修改端口映射。

### 14.8 忘记密码

```bash
# 重置密码
docker exec clawbot-gateway ./clawbot-gateway reset-password
```

---

## 附录

### A. 技术栈

| 层 | 技术 |
|----|------|
| 后端语言 | Go 1.22 |
| HTTP 框架 | Gin |
| 数据库 | SQLite |
| 前端框架 | React 19 + TypeScript |
| 构建工具 | Vite |
| 状态管理 | Zustand |
| 桌面端 | Tauri |
| 部署 | Docker |

### B. 项目结构

```
clawbot-gateway/
├── main.go                     # 入口
├── .env                        # 启动配置
├── Dockerfile                  # Docker 构建
├── docker-compose.yml          # Docker Compose
├── VISION.txt                  # 版本管理
├── AGENTS.md                   # 核心架构规则
├── docs/
│   ├── USER_GUIDE.md           # 用户指南（本文档）
│   ├── DESIGN.md               # 架构设计文档
│   ├── FRONTEND_DESIGN.md      # 前端设计文档
│   └── TEST_FRAMEWORK.md       # 测试框架文档
├── internal/
│   ├── adapter/                # 后端适配器
│   ├── api/                    # HTTP API 服务
│   ├── bot/                    # iLink 客户端
│   ├── config/                 # 配置加载
│   ├── crypto/                 # 加密工具
│   ├── database/               # 数据库层
│   ├── ilink/                  # iLink 服务端
│   ├── log/                    # 日志系统
│   ├── notify/                 # 通知模块
│   ├── route/                  # 路由引擎
│   ├── session/                # 会话管理
│   └── version/                # 版本信息
├── web/                        # 前端 SPA
│   ├── src/                    # 源码
│   │   ├── api/                # API 客户端
│   │   ├── stores/             # Zustand 状态管理
│   │   └── components/         # UI 组件
│   └── src-tauri/              # Tauri 桌面端
├── scripts/                    # 构建脚本
│   └── build-fpk.sh            # FPK 打包脚本
├── test/                       # 测试
│   ├── integration/            # 集成测试
│   ├── mock/                   # Mock 服务
│   ├── fixtures/               # 测试数据
│   └── e2e/                    # 端到端测试
└── data/                       # 运行时数据
```
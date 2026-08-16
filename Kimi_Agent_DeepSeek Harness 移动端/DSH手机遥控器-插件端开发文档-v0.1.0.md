# DSH 手机遥控器 — 插件端开发文档（v0.1.0 MVP）

> 本文档面向「电脑端 companion + 云端中继」开发者。
> 配套文档：《DSH 手机遥控器 — 移动端开发文档（v0.1.0）》。
> 版本：0.1.0 ｜ 状态：设计定稿 ｜ 目标读者：插件端 / 后端开发者

---

## 1. 项目背景与目标

DeepSeek Harness（DSH）在电脑本地以 `npx @deepseek-ai/dsh web` 运行，Web UI 监听 `127.0.0.1:3080`，官方禁止 `--host 0.0.0.0` 对外暴露。本项目的目标：让用户在手机上安全地访问自己电脑上的 DSH。

**v0.1.0 的产品目标**：用户在电脑上跑一条命令、用手机扫码配对、之后在 App 里随时打开自己电脑上的 DSH 界面。能跑通这条链路即为成功。

### 1.1 总体架构

```
┌────────────────┐    WSS 长连接（出站）   ┌────────────────┐    WSS/HTTPS    ┌────────────────┐
│ Companion CLI   │ ◄────────────────────► │  云端中继服务     │ ◄──────────────► │  手机 App       │
│ （用户电脑上）    │   无公网 IP / NAT 均可   │  Relay Server    │                 │ （WebView 壳）   │
└───────┬────────┘                        └────────────────┘                 └────────────────┘
        │ 本地 HTTP/WS 转发
        ▼
  127.0.0.1:3080（DSH Web UI）
```

三条核心原则：

1. **一切连接均为出站**：Companion 和手机都主动连云端，云端只做路由转发。用户无需公网 IP、无需配置防火墙。
2. **云端不解密业务内容**（0.2 实现端到端加密；0.1 的临时方案见 §9）。
3. **Companion 不改 DSH 任何东西**：不注入、不 patch、不要求 DSH 改监听地址，只做本地 127.0.0.1:3080 的反向代理客户端。

### 1.2 职责划分

| 组件 | 归属 | 说明 |
|---|---|---|
| Companion CLI（电脑端） | **本文档** | Node.js 命令行工具，负责配对、维持长连接、本地转发、事件上报 |
| Relay Server（云端） | **本文档** | 账号、配对、设备路由、字节转发 |
| 手机 App | 见移动端文档 | 登录、设备列表、WebView 容器 |

> 建议 Companion 与 Relay 由同一开发者实现：两者协议强耦合，分开做会产生大量对齐成本。

---

## 2. Companion CLI（电脑端）

### 2.1 技术选型

- 语言/运行时：**Node.js ≥ 18**（与 DSH 同生态，用户零额外安装）
- 分发：**npm 包**，包名占位 `dsh-remote`，用户通过 `npx dsh-remote` 直接运行
- 关键依赖：
  - `ws` — WebSocket 客户端（连云端）+ 服务端（如需要）
  - 内置 `fetch` / `http` 模块做本地转发
  - `qrcode-terminal` — 终端打印配对二维码
  - `commander` — CLI 参数解析
  - 不依赖任何原生模块（保证 Windows/macOS/Linux 免编译安装）

### 2.2 命令设计

```bash
npx dsh-remote start          # 启动并常驻：连接 DSH + 连接云端
npx dsh-remote pair           # 生成配对码并在终端打印二维码（首次 start 时自动执行）
npx dsh-remote status         # 查看连接状态（DSH 是否存活 / 云端是否在线 / 已绑定账号）
npx dsh-remote unpair         # 解除当前设备绑定
npx dsh-remote stop           # 停止常驻进程
```

`start` 可选参数：

```bash
npx dsh-remote start --dsh-port 3080 --relay wss://relay.example.com
```

### 2.3 核心职责

1. **发现 DSH**：探测 `127.0.0.1:3080` 是否存活（轮询 `GET /` 或健康端点，间隔 5s）。DSH 不在线时 Companion 仍保持云端连接，并向云端上报 `dsh_offline` 状态（App 端据此显示「电脑在线但 DSH 未启动」而不是白屏）。
2. **配对**：见 §3。
3. **维持云端长连接**：单条 WSS 连接承载所有流量（多路复用，见 §4）。断线自动重连，指数退避（1s → 2s → 5s → 10s → 30s 封顶），重连成功后重新鉴权并恢复通道。
4. **HTTP/WebSocket 转发**：收到云端的隧道消息后，打到本地 DSH 并回传结果。这是 0.1 的主流量路径（App 的 WebView 全部请求都走这里）。
5. **事件上报（预埋，0.1 必须实现发送端）**：监听 DSH 的会话事件并推送到云端。即使 App 在 0.1 不消费，协议和插件端逻辑要先就位，避免 0.2 升级时要求所有用户重装插件。见 §5。

### 2.4 本地配置与状态

配置文件：`~/.dsh-remote/config.json`

```json
{
  "deviceId": "dev_8f3a2c...",
  "deviceSecret": "…随机 32 字节 hex，仅本地存储，配对时用于换取 deviceToken",
  "deviceToken": "…配对成功后云端签发，长期凭证",
  "deviceName": "张伟的 MacBook Pro",
  "relay": "wss://relay.example.com",
  "dshPort": 3080,
  "pairedAccount": "acc_xxxx（脱敏展示用）"
}
```

- 日志：`~/.dsh-remote/logs/`，按天滚动，保留 7 天，默认只记连接状态不记业务内容。
- `deviceSecret` / `deviceToken` 文件权限 `600`。

---

## 3. 配对流程

配对的设计约束：**绑定必须发生在用户物理接触电脑的时刻**——不允许纯账号侧远程绑定新设备，否则任何拿到账号密码的人都能获得用户电脑的远程执行能力。

### 3.1 时序

```
Companion                Relay                 App
   │  1. POST /pair/session  │                   │
   │ ───────────────────────►│                   │
   │  2. {pairCode:"482913", │                   │
   │      expires: 300s}     │                   │
   │ ◄────────────────────── │                   │
   │  终端打印二维码+6位码      │                   │
   │     （用户用手机扫码）      │                   │
   │                         │  3. POST /pair/claim
   │                         │     {pairCode}     │
   │                         │ ◄───────────────── │
   │                         │  4. 校验：用户已登录、 │
   │                         │     码未过期未使用     │
   │  5. WS 推送 pair_claimed │                   │
   │ ◄────────────────────── │                   │
   │  6. POST /pair/confirm   │                   │
   │    {deviceId, deviceSecret, deviceName}      │
   │ ───────────────────────►│                   │
   │  7. {deviceToken}        │  8. 推送绑定成功     │
   │ ◄────────────────────── │ ─────────────────►│
```

### 3.2 规则

- `pairCode`：6 位数字，5 分钟有效，一次性，被 claim 后立即作废。
- 二维码内容为一个 JSON 短串：`{"v":1,"relay":"wss://relay.example.com","code":"482913"}`，App 扫码后等价于手动输入配对码。
- 一个账号可绑定多台设备；一台设备同时只能绑定一个账号（换绑需先 unpair）。
- `deviceToken`：配对确认后签发，长期有效，云端可随时吊销（用户在 App 里解绑）。Companion 之后所有云端连接凭 `deviceId + deviceToken` 鉴权。

---

## 4. 通信协议（Companion ⇄ Relay）

单条 WSS 连接，多路复用。所有消息为 JSON 文本帧，信封格式统一：

```json
{
  "v": 1,
  "type": "http_req | http_res | ws_open | ws_frame | ws_close | event | ping | pong | auth | status",
  "channel": "ch_12345",
  "id": "msg_uuid",
  "ts": 1755500000000,
  "payload": {}
}
```

- `channel`：隧道类消息（http/ws）使用，标识一条逻辑通道，Companion 按 channel 维护本地连接池。
- `event / ping / auth / status` 为控制消息，`channel` 为空。

### 4.1 鉴权

连接建立后第一条消息必须是：

```json
{"v":1,"type":"auth","id":"...","ts":0,"payload":{"deviceId":"dev_xxx","deviceToken":"..."}}
```

云端校验失败返回 `{"type":"auth_fail","payload":{"reason":"..."}}` 并关闭连接；成功返回 `auth_ok`，之后进入正常通信。

### 4.2 HTTP 隧道（App WebView 的主路径）

云端收到某个已登录用户对某台设备网页的请求后，转成隧道消息发给 Companion：

**http_req**

```json
{
  "type": "http_req",
  "channel": "ch_a1b2",
  "payload": {
    "method": "GET",
    "path": "/api/sessions?limit=20",
    "headers": {"accept": "application/json", "cookie": "..."},
    "bodyB64": ""
  }
}
```

**http_res**（Companion → 云端）

```json
{
  "type": "http_res",
  "channel": "ch_a1b2",
  "payload": {
    "status": 200,
    "headers": {"content-type": "application/json"},
    "bodyB64": "eyJvayI6dHJ1ZX0="
  }
}
```

规则：

- body 一律 base64，二进制安全。
- 单条消息 body 上限 1MB，超限分片：payload 增加 `seq` 与 `final: false` 字段，末片 `final: true`。**0.1 可以暂不支持分片，直接返回 502 并记日志**，但信封里保留字段名。
- Companion 必须剥离/重写 `Host` 头为 `127.0.0.1:3080`，其余头原样透传。

### 4.3 WebSocket 隧道（DSH 实时输出大概率走 WS/SSE，必须支持）

- `ws_open {channel, path, headers}` → Companion 向本地 DSH 建立 WS，成功回 `ws_open_ok`，失败回 `ws_close {code, reason}`。
- `ws_frame {channel, dataB64, opcode}` → 双向数据帧。
- `ws_close {channel, code, reason}` → 任一端关闭都透传到另一端。

SSE 无需特殊处理，按普通 HTTP 长响应逐段 `http_res` 分片即可（若 0.1 不实现分片，SSE 流式接口可标记为已知限制）。

### 4.4 心跳与状态

- Companion 每 25s 发 `ping`，云端回 `pong`；云端 60s 未收到任何消息则断开。
- 状态上报：DSH 存活状态变化时发送 `status {dsh: "online" | "offline"}`；云端据此维护设备展示状态。

---

## 5. 事件通道（0.1 预埋，0.2 消费）

Companion 监听 DSH 的会话事件流（DSH Web UI 本地的 WS/SSE，或轮询其会话接口，以实际可用的机制为准），将关键事件抽象为结构化消息推给云端：

```json
{
  "type": "event",
  "payload": {
    "kind": "permission_request | task_complete | task_error | session_idle",
    "sessionId": "dsh 会话 id",
    "title": "执行命令：rm -rf ./build && npm run build",
    "detail": { },
    "requiresAction": true,
    "actionId": "act_xxx（仅 permission_request）"
  }
}
```

0.1 的要求：

- **Companion 端必须实现事件采集与发送**，云端必须实现接收与暂存（写入设备最近事件表，保留最近 50 条即可）。
- App 端 0.1 不展示。这样 0.2 上推送与审批卡片时，只需 App 端和云端增量开发，存量插件无需升级。

若 DSH 当前版本没有稳定的事件接口，降级方案：Companion 解析本地 WS 流量做关键词级识别，识别不出就静默跳过——**事件功能在 0.1 允许不完整，但不允许阻塞主链路**。

---

## 6. Relay Server（云端）

### 6.1 技术选型

- Node.js（与 Companion 复用协议代码）或 Go 均可；以下按 Node.js 描述。
- 组件：REST API（认证/配对/设备管理）+ WSS 网关（设备连接与用户连接）+ 路由器。
- 存储：PostgreSQL 或 SQLite（0.1 量级 SQLite 足够）；账号、设备、配对关系、最近事件。
- 部署：单台 2C4G 起步，前置 nginx 做 TLS 终止。

### 6.2 数据模型

```
accounts(id, phone_or_email, password_hash, created_at)
devices(id, account_id, name, device_token_hash, last_seen_at, dsh_status, created_at)
pair_sessions(code, device_id, expires_at, claimed_by, used, created_at)
device_events(id, device_id, kind, payload_json, created_at)   -- 每设备保留最近 50 条
```

### 6.3 REST API

| 方法 | 路径 | 鉴权 | 说明 |
|---|---|---|---|
| POST | `/auth/register` | 无 | 注册（0.1 可用邮箱+验证码或密码，从简） |
| POST | `/auth/login` | 无 | 登录，返回 `accessToken`(JWT, 7d) + `refreshToken`(30d) |
| POST | `/auth/refresh` | refresh | 换新 accessToken |
| POST | `/pair/session` | 无（限流） | Companion 创建配对码 |
| POST | `/pair/claim` | 用户 JWT | App 认领配对码 |
| POST | `/pair/confirm` | 配对会话 | Companion 确认并换取 deviceToken |
| GET | `/devices` | 用户 JWT | 我的设备列表（含在线/DSH 状态） |
| DELETE | `/devices/:id` | 用户 JWT | 解绑并吊销 deviceToken |
| PATCH | `/devices/:id` | 用户 JWT | 重命名设备 |

### 6.4 路由与转发

- 维护两张连接表：`deviceConn[deviceId]`、`userConn[accountId]`（用户可能多端登录，值为数组）。
- App 的 WebView 请求入口：`https://relay.example.com/s/{deviceId}/*`（见移动端文档 §5）。云端鉴权（Cookie 或 query token）→ 查 `deviceConn` → 封装为 `http_req` 隧道消息 → 等待 `http_res` → 写回 HTTP 响应。超时 30s 返回 504。
- 设备离线：立即返回 503 + 错误页 JSON，App 端据此展示「电脑不在线」。
- **云端不存储、不记录任何隧道 payload 明文**，只记元数据（时间、字节数、状态码）。

### 6.5 限流与安全基线

- `/pair/session`、`/auth/*` 按 IP 限流。
- 配对码枚举防护：claim 失败 5 次锁定该账号 10 分钟。
- 全站 TLS；WSS 连接 24h 强制重连一次。

---

## 7. 建议目录结构

```
dsh-remote/
├── packages/
│   ├── companion/            # 电脑端 CLI
│   │   ├── src/
│   │   │   ├── cli.ts        # commander 入口
│   │   │   ├── config.ts     # ~/.dsh-remote 读写
│   │   │   ├── pairing.ts    # 配对流程 + 二维码
│   │   │   ├── relay-client.ts   # WSS 连接、重连、鉴权、心跳
│   │   │   ├── tunnel.ts     # channel 管理、http/ws 转发到 127.0.0.1:3080
│   │   │   ├── dsh-watch.ts  # DSH 存活探测 + 事件采集
│   │   │   └── proto.ts      # 信封与消息类型（与 relay 共享，可抽为公共包）
│   │   └── package.json      # name: dsh-remote, bin: dsh-remote
│   └── relay/                # 云端服务
│       ├── src/
│       │   ├── http-api.ts   # REST 路由
│       │   ├── gateway.ts    # WSS 接入（设备/用户）
│       │   ├── router.ts     # channel 路由、超时
│       │   ├── store.ts      # 数据层
│       │   └── proto.ts      # 同上共享
│       └── package.json
└── README.md
```

---

## 8. 0.1.0 验收标准

1. 全新电脑执行 `npx dsh-remote start`（已运行 DSH），终端打印二维码。
2. App 扫码 → 3 秒内双方提示绑定成功；`npx dsh-remote status` 显示已绑定与在线。
3. App 打开该设备，WebView 中完整加载 DSH Web UI，可正常浏览会话列表与历史。
4. WebView 内发起一次真实交互（发消息），DSH 实时流式输出在手机上可见（WS 隧道通畅）。
5. 电脑上 Ctrl+C 停掉 DSH：App 端 30s 内显示「DSH 未启动」而非白屏/超时。
6. 电脑断网 30s 恢复：Companion 自动重连，App 无需重新登录即可继续使用。
7. Companion 重启后无需重新配对（deviceToken 持久化）。
8. App 内解绑后，该设备 deviceToken 立即失效，Companion 端提示已解绑并回到待配对状态。
9. 云端日志中不出现任何业务 payload 明文。

### 已知限制（0.1 接受，需在 README 写明）

- 隧道内容非端到端加密，依赖 TLS + 云端自律（0.2 上 E2E）。
- 大于 1MB 的响应体不支持。
- SSE 长流接口可能不完整。

---

## 9. 安全说明与 0.2 预埋

0.1 的安全模型：全程 TLS；deviceToken 仅哈希落库；配对必须物理在场；云端不存明文、不记内容日志。

必须诚实面对的问题：0.1 中云端技术上「可以」解密转发内容。0.2 计划引入端到端加密（配对时 ECDH 协商会话密钥，云端退化为纯密文路由），届时协议升级路径为：信封 `v: 2` + payload 加密层。 Companion 的 `proto.ts` 从第一天起就把「信封层」与「payload 层」分开，不要把业务字段直接平铺在信封上，0.2 才能无痛升级。

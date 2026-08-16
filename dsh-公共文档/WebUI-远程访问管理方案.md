# DSH WebUI 远程访问管理方案

## 结论

可以将“添加电脑”从命令行迁移到 DSH WebUI。建议在 DSH 设置面板中注册一个顶级“远程访问”页，而不是创建第二套独立管理站。插件安装后，用户打开 `dsh web`，在该页生成配对二维码、查看已授权手机、查看近期访问会话，并撤销某一部手机的访问权。

命令行仅保留两个兜底职责：

- 打印 `dsh web` 地址和“请在 WebUI > 设置 > 远程访问中完成配对”。
- 当 WebUI 无法启动时，保留 `dsh-mobile status` / `dsh-mobile unpair` 维护命令。

## 硬约束：只通过插件 Hook 扩展 DSH

本方案只修改我们维护的 `dsh-mobile`、`dsh-relay` 和移动端，不修改、fork 或复制 DSH 源码。

允许使用的 DSH 扩展面：

- DSH bundle 的 `cordis.patch.yml` 组合层，用于挂载/卸载我们的 Host 和 Client 插件。
- Cordis `apply()` / `ctx.effect()` 生命周期 Hook，让 Companion 跟随 `dsh web` 启停。
- DSH Host webserver 路由注册 Hook，增加 `/dsh-mobile/api/*`，不改已有 `/api/*`。
- npm manifest 的 `dsh.client` 和 `./client` export，由 DSH 客户端模块加载器发现插件。
- DSH 公开的 `settings.section` slot Hook，注册“远程访问”页。

禁止的实现方式：

- 修改 `@deepseek-ai/dsh-*` 包、DSH 仓库或 `node_modules` 内文件。
- 对 DSH 编译后的 JS/CSS/HTML 做文本替换或启动时热补丁。
- 依赖私有 DOM 结构、CSS selector 或 monkey patch DSH 全局对象插入页面。
- 在 Relay 或移动端复制一份 DSH WebUI。

验收时必须证明：安装 `dsh-mobile` 后管理页出现；执行 `dsh plugin --profile web remove dsh-mobile` 并重启 DSH 后，管理页、Host 路由和 Companion 连接全部消失，DSH 本体文件哈希不变。

## 为什么不能只改一个 UI

当前 Relay 的授权模型是“一台电脑归属一个账号”。它能回答哪个账号拥有本机，但不能回答“该账号下哪一部 iPhone/Android 添加了本机”：

- access token 只含 `accountId`，没有手机实例标识。
- web ticket / WebView Cookie 只绑定 `accountId + deviceId`。
- Relay 不保存 WebView 访问会话元数据。
- 现有 `events` 表是 Companion 事件保留位，不是可审计的访问日志。

因此，“查看已添加本机的设备”需要引入手机实例和授权记录，不应使用 IP 或 User-Agent 猜测设备身份。

## WebUI 位置与实现边界

### 入口

DSH `@deepseek-ai/dsh-client-ui-settings` 对外声明了 `settings.section` 插槽，用于“一个 feature 贡献一个设置页”。`dsh-mobile` 的 client 面应注册：

```text
设置
  通用
  模型
  插件
  远程访问   <- dsh-mobile
```

这比修改主会话侧边栏稳定，也符合“安装插件后自动出现管理”的心智模型。

### 数据通道

不要使用 DSH `settingsScope` 加载远程管理数据。DSH 官方文档明确说明 settings RPC 仅限 loopback，远程浏览器会得到 unavailable。

`dsh-mobile` 应在 DSH Host webserver 上注册自己的同源路由：

```text
GET    /dsh-mobile/api/state
POST   /dsh-mobile/api/pairing
DELETE /dsh-mobile/api/pairing
DELETE /dsh-mobile/api/clients/:clientId
GET    /dsh-mobile/api/access-sessions?cursor=...
```

WebUI 只请求这些本机同源路由。Host 插件使用本地保存的 `deviceId + deviceToken` 向 Relay 调用设备管理 API；`deviceToken` 永远不返回浏览器。

### 本地管理与远程访问

同一套 DSH WebUI 也会被手机通过 Relay 访问。建议第一版权限如下：

| 功能 | 本机浏览器 | Relay 远程浏览器 |
| --- | --- | --- |
| 查看连接状态 | 允许 | 允许 |
| 查看已授权手机 | 允许 | 允许 |
| 查看访问会话 | 允许 | 允许 |
| 生成新配对码 | 允许 | 禁止 |
| 撤销手机 | 允许 | 禁止 |
| 解绑账号 | 允许 | 禁止 |

Companion 转发 Relay 流量时必须先删除浏览器传入的内部标头，再由 Companion 自己注入 `X-DSH-Mobile-Remote: 1`。Host 以此关闭远程破坏性操作。这个标头是 Companion 到 Host 的本地信任边界，不得由 Relay 或移动端直接控制。

## 页面状态

### 1. 未配对

- 状态：“远程访问未开启”。
- 主操作：“添加手机”。
- 点击后由 Host 请求 Relay `POST /pair/session`，在页面中显示二维码、6 位码和剩余时间。
- 页面轮询 Host 状态，Host 在后台执行 `pair/confirm`；确认成功后原地转入“已连接”。
- 过期后显示“重新生成”，不让安装流程阻塞 DSH 启动。

### 2. 已配对

页头显示：

- Relay 地址。
- Companion 在线 / 重连中。
- DSH 本地健康状态。
- 设备名称和 `deviceId` 短标识。
- 当前所属账号（脱敏邮箱）。

页面主体包含“已授权手机”和“近期访问”两个列表。破坏性操作放在独立的危险区，不与日常状态操作混排。

### 3. 配对中或 Relay 离线

- 配对进程是 Host 管理的可取消状态机，不再是 `apply()` 中一个长时间占用的 `pair()` Promise。
- Relay 不可用时仍能打开页面，显示最后同步的授权列表和明确的重试操作。
- 已保存的 `deviceToken` 不因一次网络失败被清除。

## 手机实例与授权模型

### 新增数据

Relay 新增：

```text
mobile_clients
  id                 随机安装实例 ID，不使用广告 ID/IMEI
  account_id
  display_name       例如“Watson 的 iPhone”，可在手机端修改
  platform           ios | android
  app_version
  created_at
  last_seen_at
  revoked_at

device_client_grants
  device_id
  client_id
  granted_at
  last_access_at
  revoked_at

access_sessions
  id
  device_id
  client_id
  account_id
  started_at
  last_seen_at
  expires_at
  ended_reason       expired | revoked | unbound | unknown
```

`mobileClientId` 由 App 首次启动时生成 UUID，保存在 secure storage。不采集 IMEI、IDFA、精确位置、通讯录或硬件序列号。

### 认证与配对变更

- `POST /auth/register` / `POST /auth/login` 增加可选 `client` 字段：`{ id, displayName, platform, appVersion }`。
- Relay 签发的 access token 增加 `clientId`；refresh token 记录也绑定 `clientId`。
- 首次绑定继续使用 bootstrap `POST /pair/session`；`POST /pair/claim` 在确定 owner account 的同时为扫码手机创建第一条 `device_client_grants`。
- 电脑已绑定后再添加手机，必须使用设备鉴权的 grant pair session，它绑定已有 `deviceId`，不生成新电脑，不轮换 Companion `deviceToken`。
- grant pair 只允许同一 owner account 下的手机 claim。跨账号共享不在本版范围内。
- `POST /web-ticket` 除检查 account 拥有设备，还必须检查当前 `clientId` 的 grant 未被撤销。
- web ticket 和 WebView Cookie 内部携带 `clientId + accessSessionId`，但这两个值不暴露给 DSH 业务页面。
- 旧版未携带 `clientId` 的 token 只允许完成 refresh 迁移，生成一个 legacy client；不得长期保留无设备身份的全账号访问。

### 撤销语义

“移除手机”只撤销该 `clientId` 对本机的 grant，并立即失效其本机 web sessions；不删除用户账号，不影响该手机访问账号下其他电脑，也不解绑本机的 owner account。Relay 在每一次根路径 HTTP/WebSocket Cookie 授权时都要检查 `accessSessionId` 和 grant 仍有效，不能只依赖已签名 Cookie 的过期时间。

## Relay 设备管理 API

Companion 使用设备凭证调用，不使用用户 access token：

```text
GET    /device-management/:deviceId
GET    /device-management/:deviceId/access-sessions?cursor=&limit=50
POST   /device-management/:deviceId/pair-sessions
DELETE /device-management/:deviceId/clients/:clientId
POST   /device-management/:deviceId/unbind
```

请求头建议使用 `Authorization: Device <deviceToken>`。Relay 对 token 做哈希比对，验证 token 必须属于 path 中的 device。所有返回值都不含 token hash、refresh token、Cookie、访问正文或 DSH URL path。

`GET /device-management/:deviceId` 返回示例：

```json
{
  "device": {
    "id": "dev_xxx",
    "name": "Watson's MacBook Air",
    "ownerEmailMasked": "w***@example.com",
    "connected": true,
    "dshStatus": "online"
  },
  "clients": [{
    "id": "mob_xxx",
    "displayName": "Watson 的 iPhone",
    "platform": "ios",
    "appVersion": "0.1.0",
    "grantedAt": 1786860000000,
    "lastAccessAt": 1786860100000
  }]
}
```

`POST /device-management/:deviceId/pair-sessions` 生成的二维码仍使用 v1 外形，但 Relay 内部将 code 标记为 `grant` 类型并绑定现有 device。Mobile 继续调用 `POST /pair/claim`，Relay 按 code 类型执行 bootstrap claim 或 grant claim，从而不需要移动端维护两套扫码协议。

## 访问日志的定义

第一版只记录“访问会话”，不记录每一个 HTTP 请求。

### 记录

- 哪一部已授权手机。
- 开始时间、最后活动时间、过期时间。
- 会话是否因撤销/解绑失效。
- 平台和 App 版本。

### 不记录

- DSH 任务文本、模型输出、工具调用、文件路径。
- HTTP body、WebSocket frame、Authorization、Cookie、ticket。
- 完整 URL 或 query string。
- 精确 IP。如未来出于安全需求保留 IP，需要独立的隐私评审和保留期。

Relay 在消费 web ticket 时创建 `access_sessions`，对同一 session 的 `last_seen_at` 最多每 60 秒更新一次，避免每个 DSH 资源请求写 SQLite。日志建议保留 30 天，每台电脑最多 500 条，以先到期为准。

## Companion 重构

当前 `plugin.ts` 在没有 `deviceToken` 时直接 `await pair(config)`，`pair()` 负责终端输出、轮询和持久化。WebUI 方案需将其拆成一个明确的状态服务：

```text
RemoteAccessService
  snapshot()
  beginPairing()
  cancelPairing()
  listClients()
  listAccessSessions(cursor)
  revokeClient(clientId)
  unbind()
```

服务状态：`unpaired | pairing | connecting | online | relayOffline | dshOffline`。配对是可取消 effect；DSH 关闭时中断 confirm 轮询，已启动的 RelayClient 和管理缓存一起释放。

不再要求首次 `dsh web` 在终端必须可见。无凭证时插件正常启动，只将状态设为 `unpaired`；WebUI 点击后才创建 pair session。

## Client 插件打包

`dsh-mobile` 包新增 `./client` export 和 `dsh.client` manifest，让 `@deepseek-ai/dsh-client-modules` 自动将其加入 `/plugins/dsh-mobile/client.js`。client 面依赖 DSH 已有的 React/runtime/locale/settings/slots 作为 peer，不在包内复制 React。

建议 manifest 注入：

```json
{
  "dsh": {
    "client": {
      "platform": "web",
      "inject": [
        "@deepseek-ai/dsh-client-runtime",
        "@deepseek-ai/dsh-client-locale",
        "@deepseek-ai/dsh-client-ui-settings"
      ]
    }
  }
}
```

页面使用 `settings.section` 注册，完整管理数据通过 `/dsh-mobile/api/*` 获取，不把 Relay 设备 token 放入 client bundle、DOM、localStorage 或 DSH settings document。

## 分阶段实施

### 阶段 A：WebUI 配对入口

只改插件：

- 加入“远程访问”设置页。
- 将配对改为状态服务，二维码在 WebUI 渲染。
- 显示 Relay/Companion/DSH 状态。
- 终端 QR 从主流程移除，保留维护 CLI。

这一阶段不改授权模型，可先独立发布。

#### 阶段 A 已实现接口（2026-08-16）

插件当前已实现以下同源 Host API：

```text
GET    /dsh-mobile/api/state
POST   /dsh-mobile/api/pairing
DELETE /dsh-mobile/api/pairing
```

`GET /state` 返回当前电脑、Relay、DSH、配对状态，以及 `localActionsAllowed`。配对中会额外返回六位码、失效时间、v1 QR payload 和由 Host 生成的 SVG；不会返回 `deviceSecret` 或 `deviceToken`。

```json
{
  "phase": "unpaired",
  "deviceId": null,
  "deviceName": "Watson's Computer",
  "relay": "https://dsh-relay-production.up.railway.app",
  "dsh": "online",
  "relayConnection": "offline",
  "pairing": null,
  "localActionsAllowed": true
}
```

Companion 对所有 Relay 转发请求强制覆盖 `X-DSH-Mobile-Remote: 1`。带此标头的 `POST` / `DELETE` 返回 `403` 和 `reason: local_management_required`；`GET /state` 保持可读。阶段 A 页面每两秒读取状态，首次安装无 token 时不会自动创建配对会话，也不会阻塞 `dsh web`。

阶段 A 尚未实现已授权手机列表、单手机撤销、访问会话日志和账号解绑。这些控件不会在页面中伪造展示，必须等待阶段 B/C 的 Relay 与 Mobile 授权模型完成。

### 阶段 B：手机实例与撤销

同时改 Relay + Mobile + Plugin：

- Mobile 生成安装实例 ID，登录/刷新携带 client 身份。
- Relay 增加 `mobile_clients` 和 `device_client_grants`。
- WebUI 显示已授权手机，并支持单独撤销。
- 撤销后相关 web session 立即失效。

### 阶段 C：访问会话日志

- Relay 增加 `access_sessions` 和限频 last-seen 更新。
- WebUI 增加时间、手机、平台、会话结果列表。
- 实施 30 天 / 500 条清理策略。
- 验证 Relay 日志和 SQLite 都不包含 DSH 业务 payload。

#### 阶段 C MVP 接口契约

在移动端尚未提供稳定 `mobileClientId` 前，Relay 在 web ticket 首次消费时创建访问会话，并从 WebView User-Agent 中只提取以下低敏字段：

- `platform`：`ios | android | other`。
- `deviceLabel`：例如 `iPhone`、`iPad`、Android 机型或“移动设备”。
- `osVersion`：仅保留系统版本；无法可靠识别时为 `null`。

Relay 不保存原始 User-Agent、IP、请求路径、query、请求头或 DSH payload。每个 web ticket 对应一条 session；同一 session 的 `lastSeenAt` 最多每 60 秒写入一次，数据保留 30 天且每台电脑最多 500 条。

Companion 使用设备凭证读取：

```text
GET /device-management/:deviceId/access-sessions?limit=50
Authorization: Device <deviceToken>
```

返回：

```json
{
  "sessions": [{
    "id": "access_xxx",
    "deviceLabel": "iPhone",
    "platform": "ios",
    "osVersion": "18.6",
    "startedAt": 1786870000000,
    "lastSeenAt": 1786870060000,
    "expiresAt": 1786877200000,
    "status": "active"
  }]
}
```

Host 同源暴露 `GET /dsh-mobile/api/access-sessions`，但不把设备凭证交给浏览器。`status` 仅为 `active | expired | ended`。后续 Mobile 提供稳定实例 ID 后，可以增量加入 `mobileClientId` 和用户命名，不改变现有字段语义。

## 验收标准

1. 新 profile 安装 `dsh-mobile` 后，不看终端 QR 也能在 WebUI 完成首次配对。
2. 配对过程不阻塞 DSH 首页、任务和其他插件加载。
3. 配对成功后 WebUI 显示手机实例名、平台、授权时间和最后访问时间。
4. 撤销一部手机后，它无法获取新 web ticket，已有 WebView 会话也被拒绝；其他手机不受影响。
5. 远程 WebUI 可查看但不能生成配对码、撤销手机或解绑本机。
6. 访问日志只含会话元数据，不含 prompt、response、tool、path、header、Cookie 或 token。
7. Companion 离线、Relay 离线、配对码过期、撤销当前手机等状态都有可恢复 UI。

## 已知风险

- DSH 当前为 `0.1.0-rc.6`，`settings.section` 虽是公开插槽，仍需在 DSH 升级时做客户端插件兼容验证。
- 引入 per-client grant 后，旧版移动端需要迁移或强制升级，否则无法准确撤销某一部手机。
- Relay 仍是单实例 SQLite；日志增长需要上限、索引和清理任务。
- “远程管理本机授权”是高风险能力，首版保持远程只读，不应为了操作便利放开撤销/解绑。

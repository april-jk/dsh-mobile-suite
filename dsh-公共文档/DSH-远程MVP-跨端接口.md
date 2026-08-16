# DSH 远程 MVP 跨端接口

本文件是移动端、电脑端 Companion 和 Relay 的共同契约。业务 payload 不写入 Relay 日志。

## 地址约定

- 本地开发 Relay：`http://127.0.0.1:8787`
- 生产 Relay：Railway HTTPS 域名，例如 `https://dsh-relay.example`
- Companion 设备连接：把 Relay 地址的协议替换为 `ws` / `wss`，路径 `/device`
- WebView 入口：`GET /s/{deviceId}/{path}`

## 认证

`POST /auth/register` `{ email, password }`，密码至少 8 位，返回 `{ accessToken, refreshToken }`。

`POST /auth/login` 同上。`POST /auth/refresh` `{ refreshToken }` 会轮换 refresh token。Access token 使用 `Authorization: Bearer <token>`，有效期 7 天；refresh token 有效期 30 天。

## 配对

1. Companion `POST /pair/session`，得到 `{ code, deviceId, deviceSecret, expiresAt }`。deviceSecret 只保存在电脑本地。
2. Companion 将 `{"v":1,"relay":"...","code":"..."}` 编码为二维码，同时显示六位 code。
3. 手机带用户 JWT 调 `POST /pair/claim` `{ code }`。
4. Companion 每 2 秒调用 `POST /pair/confirm` `{ deviceId, deviceSecret, deviceName }`。未认领时返回 `202 { status: "pending" }`，认领后返回 `{ deviceToken }`。
5. Companion 之后只使用 `deviceId + deviceToken` 建立 `/device` WSS 连接。

配对码 5 分钟有效、一次性；设备解绑后 deviceToken 立即失效。

## 设备管理

- `GET /devices`：返回 `{ devices: [{ id, name, online, dshStatus, lastSeenAt }] }`
- `PATCH /devices/:id` `{ name }`
- `DELETE /devices/:id`
- `POST /web-ticket` `{ deviceId }`：返回一次性 60 秒 ticket

WebView 第一次请求使用 `GET /s/{deviceId}/?ticket=<ticket>`。Relay 校验后设置 `HttpOnly; SameSite=Lax; Path=/` Cookie（有效 2 小时），后续请求和 WebSocket 升级均使用 Cookie。`Path=/` 用于承载 DSH 生成的 `/assets`、`/plugins`、`/api` 等绝对路径。ticket 只能使用一次。

## 隧道信封

Companion 与 Relay 的每一帧都是 JSON：

```json
{
  "v": 1,
  "type": "auth | auth_ok | http_req | http_res | http_close | ws_open | ws_open_ok | ws_frame | ws_close | status | ping | pong | event",
  "channel": "ch_uuid",
  "id": "message_uuid",
  "ts": 1755500000000,
  "payload": {}
}
```

设备连接建立后的第一帧必须是：

```json
{"v":1,"type":"auth","id":"...","ts":0,"payload":{"deviceId":"dev_x","deviceToken":"..."}}
```

HTTP 请求 payload：`{ method, path, headers, bodyB64 }`；响应 payload：`{ status?, headers?, bodyB64, seq, final }`。body 使用 base64，响应按 `seq` 递增分片，`final:false` 保持连接，`final:true` 结束，因此支持 SSE 和长响应。`Host`、`Origin`、`Referer` 由 Companion 重写为本地 DSH 地址。浏览器中断请求时 Relay 发送同 channel 的 `http_close`，Companion 取消本地请求。

WebSocket 使用同一 `channel`：Relay 发 `ws_open`，双方互发 `ws_frame`，任一侧关闭时发 `ws_close`。不要在移动端拦截 WebSocket。

状态帧：`status { dsh: "online" | "offline" }`。Relay 无设备连接返回 `503 {"reason":"device_offline"}`；设备在线但 DSH 探测失败返回 `503 {"reason":"dsh_offline"}`。

## MVP 已知限制

- Relay 暂未提供端到端加密，依赖 TLS/WSS；这是公开 Beta 的已披露限制。
- Relay 为单实例，SQLite 需要 Railway 持久卷。
- HTTP 请求体仍由 Relay 在转发前缓冲；HTTP 响应与 SSE 已支持分片流式转发。
- 事件只保留协议和最近 50 条存储，不做 DSH WebSocket 内容推断。

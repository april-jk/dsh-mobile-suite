# DSH 远程 MVP 跨端接口

本文件是移动端、电脑端 Companion 和 Relay 的共同契约。业务 payload 不写入 Relay 日志。

## 地址约定

- 本地开发 Relay：`http://127.0.0.1:8787`
- 生产 Relay：Railway HTTPS 域名，例如 `https://dsh-relay.example`
- Companion 设备连接：把 Relay 地址的协议替换为 `ws` / `wss`，路径 `/device`
- WebView 入口：Mobile 本机 `http://127.0.0.1:<ephemeral>/{path}`；Relay 仅提供 `/client-tunnel` 密文 WSS

## 认证

`POST /auth/register` `{ email, password }`，密码至少 8 位，返回 `{ accessToken, refreshToken }`。

`POST /auth/login` 同上。`POST /auth/refresh` `{ refreshToken }` 会轮换 refresh token。Access token 使用 `Authorization: Bearer <token>`，有效期 7 天；refresh token 有效期 30 天。

## 配对

1. Companion `POST /pair/session`，得到 `{ code, deviceId, deviceSecret, expiresAt }`。deviceSecret 只保存在电脑本地。
2. Companion 本地生成 32-byte master key，将 `{"v":2,"relay":"...","code":"...","e2eeKey":"<base64url>"}` 编码为二维码；master key 不上传 Relay，生产配对不接受单独的六位 code。
3. 手机带用户 JWT 调 `POST /pair/claim` `{ code }`。
4. Companion 每 2 秒调用 `POST /pair/confirm` `{ deviceId, deviceSecret, deviceName }`。未认领时返回 `202 { status: "pending" }`，认领后返回 `{ deviceToken }`。
5. Companion 之后只使用 `deviceId + deviceToken` 建立 `/device` WSS 连接。

配对码 5 分钟有效、一次性；设备解绑后 deviceToken 立即失效。

## 设备管理

- `GET /devices`：返回 `{ devices: [{ id, name, online, dshStatus, lastSeenAt }] }`
- `PATCH /devices/:id` `{ name }`
- `DELETE /devices/:id`
- `POST /device-management/:id/unbind` ：Companion 使用 `Authorization: Device <deviceToken>` 在电脑本机移除整机配对，Relay 撤销设备令牌、结束访问会话并断开当前连接
- `POST /web-ticket` `{ deviceId }`：返回一次性 60 秒 ticket

Mobile 用一次性 ticket 在 `Authorization: WebTicket` 中连接 `/client-tunnel`，随后启动仅绑定 loopback 的本地 HTTP/WS 网关。WebView 只访问本地网关；HTTP、SSE 和 WebSocket 作为加密 inner envelope 转发。ticket 只能使用一次且不得出现在 URL。

## 隧道信封

Companion、Mobile 与 Relay 的 outer frame 都保持 v1 JSON；业务信封按 `Tunnel-E2EE-v1.md` 加密：

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

解密后的 HTTP 请求 payload 为 `{ method, path, headers, bodyB64 }`；响应 payload 为 `{ status?, headers?, bodyB64, seq, final }`。响应按 `seq` 递增分片，因此支持 SSE 和长响应。`Host`、`Origin`、`Referer` 由 Companion 重写为本地 DSH 地址。Mobile 本地浏览器中断请求时发送加密 `http_close`，Companion 取消本地请求。

WebSocket 使用同一加密 inner `channel`：Mobile 本地网关发 `ws_open`，双方互发 `ws_frame`，任一侧关闭时发 `ws_close`。Relay 只看到 sealed frame。

状态帧：`status { dsh: "online" | "offline" }`。Relay 无设备连接返回 `503 {"reason":"device_offline"}`；设备在线但 DSH 探测失败返回 `503 {"reason":"dsh_offline"}`。

## MVP 已知限制

- 业务内容采用 Mobile 到 Companion 的端到端加密；Relay 仍可见账号、设备、在线状态、密文长度和时序。
- Relay 为单实例，SQLite 需要 Railway 持久卷。
- HTTP 请求体由 Mobile loopback 网关在加密前缓冲；HTTP 响应与 SSE 以 sealed frame 分片流式转发，Relay 不见明文。
- Relay 事件表最多保留最近 50 条 `kind` 元数据，`payload_json` 固定为空对象；业务事件正文必须走 sealed data plane。

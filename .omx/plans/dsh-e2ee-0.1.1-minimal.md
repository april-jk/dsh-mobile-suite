# DSH 0.1.1 端到端加密最小实施方案

## 结论

当前远程转发只有 HTTPS/WSS 链路加密，不是端到端加密。真正的最小修复必须同时改 Mobile、Relay 和 Companion：WebView 改连 Mobile 进程内的 loopback 网关，Mobile 与 Companion 之间传输 AEAD 密文，Relay 只认证、限流并盲转发。

本方案保留现有 tunnel outer envelope `v: 1`，新增互不兼容的密文消息类型并强制能力协商；二维码独立升级为 `v: 2`。业务 HTTP/WS 的现有 v1 信封作为密文内层继续复用。

## 当前状态

- Mobile WebView 直接加载 Relay `/s/{deviceId}`，Relay 是浏览器明文 HTTP 的终止点：`dsh-mobile/lib/features/session/session_webview_page.dart:59`、`:298`。
- Relay 明文读取 method/path/headers/body，明文组装 status/headers/body 和 WebSocket frame：`dsh-relay/src/server.ts:488`、`:638`、`:745`。
- Companion 的 envelope 是明文 JSON，HTTP/WS 内容只做 base64：`dsh-plugin/src/relay-client.ts:6`、`:92`、`:153`、`:222`。
- 当前 QR 只有 `v/relay/code`；`deviceSecret` 由 Relay 生成，不能作为 Relay 不可知的 E2EE key：`dsh-plugin/src/pairing.ts:12`、`dsh-plugin/src/remote-access.ts:210`。
- 公共契约明确披露当前无 E2EE：`dsh-公共文档/DSH-远程MVP-跨端接口.md:65`。
- Android 已只对 loopback 放行明文，iOS 已允许本地网络：`dsh-mobile/android/app/src/main/res/xml/network_security_config.xml:2`、`dsh-mobile/ios/Runner/Info.plist:31`。
- 版本号已发生占用：suite tag `v0.1.1`、Companion tag `v0.1.2` 已存在；Mobile 为 `0.1.1+2`，Relay 为 `0.1.1`。不能覆盖既有 tag 或重复发布包版本。

## 威胁模型

### 必须防护

- Relay 操作者、Relay 进程或 Relay 数据库读取 DSH 请求和响应内容。
- Relay 或链路攻击者篡改、重放、跨会话搬运业务帧。
- 旧客户端或缺失密钥时静默降级到明文 `/s/{deviceId}`。
- 本机其他进程未经授权访问 Mobile 的 loopback 网关。

### 明确不防护

- Mobile 或 Companion 端点自身被攻陷。
- Relay 观察账号、deviceId、在线状态、连接时间、密文长度和时序。
- 0.1.1 不提供前向保密：设备主密钥日后泄露时，已被记录的密文可能被追溯解密。X25519 authenticated ephemeral handshake 留给后续协议版本。

## 设计决策

### 1. 安全配对

Companion 在开始配对时本地生成 32-byte `e2eeMasterKey`，不得上传 Relay。QR 改为：

```json
{"v":2,"relay":"https://...","code":"482913","e2eeKey":"<base64url-32-bytes>"}
```

Mobile 扫码后把 key 暂存在内存；`POST /pair/claim` 成功返回 `deviceId` 后，按账号和 deviceId 写入 `flutter_secure_storage`。Companion 在 `/pair/confirm` 成功后写入现有 0600 配置文件。解绑、退出账号或重新配对时删除 key。

六位手输码不携带高熵秘密，生产 0.1.1 不允许仅凭手输码完成 claim；只接受 QR v2 安全配对。手输入口只能在显式 legacy 开发模式保留，不实现基于六位码的自制密钥派生。

### 2. 会话握手与密钥派生

Mobile 用现有 bearer token 请求一次性 web ticket。Relay 返回 additive 字段 `tunnelUrl`、`accessSessionId` 和 `e2eeRequired: true`。Mobile 使用 `Authorization: WebTicket <ticket>` 连接 `/client-tunnel`，避免 ticket 出现在 URL 和访问日志。

Mobile 与 Companion 使用二维码主密钥完成 PSK challenge-response：

1. Mobile 发送随机 32-byte `clientRandom` 和 HMAC-SHA256 client proof。
2. Companion 验证后生成随机 32-byte `serverRandom` 并返回 transcript HMAC。
3. 双方用 HKDF-SHA256 从 master key、两个 random 和 accessSessionId 派生 `c2dKey`、`d2cKey` 及各自 32-bit nonce prefix。
4. 任一 proof 失败、超时或 key 缺失都关闭会话，不允许明文 fallback。

### 3. 密文帧

outer envelope 继续使用 `v: 1`，新增 `client_open`、`device_accept`、`sealed`、`client_close`。`sealed` 只暴露：

```json
{
  "v": 1,
  "type": "sealed",
  "id": "...",
  "ts": 0,
  "payload": {
    "accessSessionId": "...",
    "seq": "0",
    "ciphertextB64": "..."
  }
}
```

- 算法：AES-256-GCM。
- 每方向独立 key 和 nonce prefix；nonce 为 `prefix32 || seq64`。
- `seq` 使用十进制字符串，避免 JavaScript number 精度问题；必须从 0 严格递增。
- AAD 为确定性 UTF-8 JSON array：`["dsh-e2ee",1,accessSessionId,direction,seq]`。
- GCM tag 随 ciphertext 一起编码。重复、乱序、认证失败或跨会话 frame 立即关闭 secure session。
- 解密后的 plaintext 是现有 `http_req/http_res/http_close/ws_open/ws_open_ok/ws_frame/ws_close` envelope；channel、path、headers、status 和内容均不出密文边界。
- `auth/auth_ok/status/ping/pong` 仍可由 Relay 看见，但不得携带业务正文；`event` 在 0.1.1 禁止正文持久化，正文事件必须走 sealed data plane。

### 4. Mobile loopback 网关

Mobile 为每个 WebView session 启动一个仅绑定 `127.0.0.1` 随机端口的 `dart:io HttpServer`：

- WebView 首次加载带 128-bit 本地 bootstrap capability 的 URL；网关消费后设置 `HttpOnly; SameSite=Strict; Path=/` cookie 并重定向到 `/`。
- 每个 HTTP 和 WebSocket 请求都验证 capability cookie、精确 Host 和 session 状态。
- HTTP 请求映射到 inner `http_req`；响应分片直接写回，保留 SSE 流式语义。
- WebSocket upgrade 映射到 inner `ws_*`，不做 JavaScript shim。
- DSH 的 `/assets`、`/plugins`、`/api` 和根 WebSocket 保持同一 localhost origin。
- 页面关闭、App 进入终止态或 tunnel 失败时关闭所有 channel、WebSocket、HttpServer 并擦除会话 key。

### 5. Relay 盲转发

Relay 新增 `/client-tunnel` WebSocket endpoint：

- 验证并单次消费 web ticket，绑定 account/device/accessSessionId。
- 只在对应 Companion capability 为 `sealed-tunnel-v1` 时建立路由。
- 双向原样转发握手和 sealed frames，不解析 plaintext、不拥有 E2EE key。
- 继续限制每设备/全局 secure session 数、frame ciphertext 大小、总字节、速率、空闲时间和握手时间。
- 原先依赖 plaintext channel 的 per-HTTP/WS limits 移到 Mobile/Companion；Relay 只执行 session/byte/frame 级资源限制。
- `/s/{deviceId}` legacy proxy 由 `ALLOW_LEGACY_WEB_PROXY=false` 控制，生产默认关闭。缺能力返回 `426 {"reason":"e2ee_required"}`，不得自动回退。

## 最小修改范围

### 公共契约先行

1. 新建 `dsh-公共文档/Tunnel-E2EE-v1.md`：威胁模型、QR v2、握手、HKDF、AEAD、AAD、序号、错误码、限制和测试向量。
2. 更新 `dsh-公共文档/API-v1.md:59`、`:126`：QR v2、安全扫码要求、web-ticket additive 响应、`/client-tunnel` 和 `e2ee_required/e2ee_handshake_failed`。
3. 更新 `dsh-公共文档/Tunnel-Protocol-v1.md:5`：声明 inner legacy envelope 与 sealed outer message 的关系；outer 仍为 v1。
4. 更新 `dsh-公共文档/DSH-远程MVP-跨端接口.md:38` 和 `Integration-Plan.md:14`：移除 TLS-only release gate，加入密文、篡改、重放、降级和重新配对验收。

### Companion

1. 新增 `dsh-plugin/src/e2ee.ts`：随机 key、HMAC/HKDF、AES-GCM、seq/AAD 和错误类型，全部使用 `node:crypto`，不新增 npm 依赖。
2. 扩展 `dsh-plugin/src/config.ts:5`，保存 `e2eeMasterKey` 和 capability；沿用目录 0700、文件 0600。
3. 同步修改 `dsh-plugin/src/pairing.ts:12` 和 `dsh-plugin/src/remote-access.ts:155` 两条配对路径，生成同一 QR v2 语义；解绑清 key。
4. 在 `dsh-plugin/src/relay-client.ts:65` auth 中声明 `sealed-tunnel-v1`；在 `:92` 和 `:122` 集中处理 secure session，解密后复用现有 `http()`/`openWs()`。

### Relay

1. 在 `dsh-relay/src/server.ts:366` 扩展 web-ticket 响应并绑定 E2EE capability。
2. 在 `dsh-relay/src/server.ts:615` 增加 `/client-tunnel` upgrade、ticket 鉴权、双端 route 和 opaque frame relay。
3. 复用 `dsh-relay/src/limits.ts` 的限流模式，新增 secure session/frame/byte 边界和 teardown。
4. 将 `dsh-relay/src/server.ts:488` 的 legacy HTTP/WS proxy 放在 off-by-default 配置后；生产启动配置必须拒绝显式明文 fallback。

### Mobile

1. 新增 `dsh-mobile/lib/data/device_key_store.dart`，复用现有 `flutter_secure_storage`，按 account/deviceId 保存和删除 master key。
2. 修改 `dsh-mobile/lib/features/pairing/pair_payload_parser.dart:6` 支持 QR v2；修改 claim 流程在 claim 成功后原子关联 key。
3. 新增 `dsh-mobile/lib/features/session/e2ee_codec.dart` 和 `secure_tunnel.dart`，实现 handshake、HKDF、AES-GCM、seq、重连和 fail-closed。
4. 新增 `dsh-mobile/lib/features/session/local_session_proxy.dart`，处理 HTTP、SSE、WebSocket、capability cookie 和资源清理。
5. 修改 `dsh-mobile/lib/features/session/session_webview_page.dart:59`、`:298`，从 Relay URL 改为 local proxy URL；修改 `session_policy.dart:53` 的 origin policy。
6. Mobile 需要一个经过审计的 AEAD/HKDF 实现。Dart 标准库没有该能力，建议唯一新增依赖为 `cryptography`；执行前需明确接受这一依赖，不手写密码算法。

## 实施顺序

1. 先合入公共契约、固定跨语言 known-answer vectors 和 fail-closed 规则。
2. 实现 Companion crypto codec 与单元测试，再实现 Mobile 同向量 codec；两端先做离线互操作测试。
3. 实现 Relay `/client-tunnel` opaque router 和资源边界测试。
4. 实现 Mobile loopback HTTP/SSE/WS 网关并对接 secure tunnel。
5. 完成三端本地 E2E，随后部署 Relay dual-path，但 legacy 仅限显式开发开关。
6. 发布新组件版本，强制旧设备重新扫码；确认采用率后生产关闭 `/s` legacy path。

## 验收标准

- Relay 进程抓到的 frame、日志和 SQLite 中不出现 canary path、header、prompt、response 或 WebSocket 内容。
- 修改 ciphertext、tag、AAD、seq、sessionId 或方向后，接收端 1 frame 内关闭会话，业务请求不抵达 DSH。
- 重放旧 frame、复用旧 ticket、跨 session 搬运 frame 全部失败。
- key 缺失、QR v1、Companion 缺 capability、旧 Mobile 均返回明确升级/重新扫码状态，不进入 `/s` 明文路径。
- HTTP 页面、绝对资源、POST、分片响应、SSE、WebSocket、取消请求和重连全部通过真实 DSH 端到端测试。
- Mobile loopback 只监听 `127.0.0.1`；无 capability cookie、Host 错误、session 关闭后的访问均被拒绝。
- 解绑、退出账号和重新配对后旧 key 失效；旧密文 session 立即关闭。
- Companion 执行 build/test，Relay 执行 build/test，Mobile 执行 format/analyze/test；再跑 Railway cloud smoke test。

## 版本与发布

“0.1.1”只能作为当前产品安全里程碑名称，不能重写已存在的 suite `v0.1.1` 或 Companion `v0.1.2` tag。建议实际不可变发布号：

- Suite：`v0.1.2`
- Companion npm：`0.1.3`
- Relay：`0.1.2`
- Mobile：`0.1.2+3`

若业务必须继续称“0.1.1”，发布说明可写“0.1.1 E2EE completion”，但仓库 tag/package version 仍使用上述递增版本。

## 剩余风险

- PSK 方案没有前向保密；若这不被接受，必须在本版本增加 authenticated ephemeral X25519，协议和跨语言测试范围会扩大。
- Mobile loopback proxy 是本次最大的新代码面，尤其要实测 iOS/Android WebSocket、SSE、cookie 和后台恢复。
- 关闭 legacy proxy 会要求所有现有设备重新扫码，必须通过 minimum mobile version 和清晰迁移提示一起发布。
- Relay 仍掌握账号/设备关系与流量元数据；文案只能声称“业务内容端到端加密”，不能声称完全匿名。

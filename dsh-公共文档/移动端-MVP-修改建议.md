# DSH Mobile MVP 修改建议

本文档用于将当前 Flutter 客户端修正到已部署、已验证的 DSH 远程 MVP 链路。客户端整体结构可以保留：登录、安全存储、扫码配对、设备列表和 InAppWebView 已经存在。当前主要问题不是重做 UI，而是默认运行模式和异常状态与真实 Relay 不一致。

## 已确认的服务端基线

- 生产 Relay：`https://dsh-relay-production.up.railway.app`
- 插件名：`dsh-mobile`，随 `dsh web` 自动启停 Companion。
- 配对二维码：`{"v":1,"relay":"https://...","code":"123456"}`。
- WebView 入口：`GET /s/{deviceId}/?ticket={singleUseTicket}`。
- Relay 设置 `HttpOnly; SameSite=Lax; Secure; Path=/` Cookie。`Path=/` 不可缩小，因为 DSH 会请求根路径 `/assets`、`/plugins`、`/api` 和 WebSocket。
- 已在 Railway 真实链路验证：登录、配对认领、设备在线、加载完整 DSH UI、浏览电脑目录、`workspace.create`、`session.create` 和 `session.prompt` 均成功。

## P0：MVP 发布前必须修改

### 1. 默认连接真实 Railway，不得默认 Mock

当前 `lib/core/app_config.dart` 的 `DSH_USE_MOCK` 默认值为 `true`，Relay 默认值为 `http://127.0.0.1:8787`。这会使普通 APK/IPA 始终进入演示数据，手机上的 `127.0.0.1` 也不是电脑。

修改为：

```dart
relayBaseUrl: String.fromEnvironment(
  'DSH_RELAY_URL',
  defaultValue: 'https://dsh-relay-production.up.railway.app',
),
useMock: bool.fromEnvironment('DSH_USE_MOCK', defaultValue: false),
```

Mock 只允许在开发命令中显式启用：`--dart-define=DSH_USE_MOCK=true`。Release 构建应在启动时拒绝非 HTTPS Relay；debug 构建可保留 loopback 例外。

### 2. 失效 refresh token 必须回到登录页

`lib/data/relay_service.dart` 的 401 拦截器在 refresh 失败后只返回原始错误，但 `AuthController` 仍保持 `authenticated`。结果是用户停留在设备页反复看到错误，无法自动恢复。

建议将“会话失效”作为明确事件传给 `AuthController`：清除 secure storage，将状态设为 `unauthenticated`，显示“登录已过期”。并发 401 仍使用现有 `_refreshInFlight` 合并，不要多次轮换 refresh token。

### 3. WebView 503 不能使用进页时的旧设备状态判断

`lib/features/session/session_webview_page.dart` 当前使用 `widget.device.online` 区分 `device_offline` 与 `dsh_offline`。该对象是进入页面时的快照，电脑在打开页面后断线时会被错误显示为“DSH 未启动”。

收到主文档 503 时，重新请求 `GET /devices`并按最新 `online` / `dshStatus` 显示状态。查询也失败时显示中性的“暂时无法连接”，不要猜测原因。`504 tunnel_timeout` 应显示可重试的网络故障。

### 4. 端到端验收必须关闭 Mock

不能用 `MockRelayService` 作为 MVP 完成证据。至少在 iOS 和 Android 各完成一次：

1. 用新邮箱注册并退出/重新登录。
2. 扫描电脑终端中 `dsh-mobile` 显示的二维码。
3. 看到同一 `deviceId` 出现且为在线。
4. 打开设备，确认 DSH 页面的 `/assets`、`/plugins`、`/api` 和 WebSocket 无 401/404。
5. 在 DSH 中选择电脑目录，新建会话并发送指令。
6. 将 DSH 停止后看到“DSH 未启动”；将 Companion 停止后看到“电脑离线”。
7. 使会话 Cookie 失效，确认客户端获取新 ticket 且不循环刷新。

## P1：公开 MVP 建议一并修改

### 5. 配对确认改为可恢复的等待状态

`DeviceController.waitForDevice()` 只等待 8 秒。Companion 本身以 2 秒周期 confirm，再叠加移动网络、Railway 冷启动或 App 切后台，8 秒容易假失败。

建议 claim 成功后立即进入“等待电脑确认”页，最多自动等待 30-60 秒，使用 1/2/3/5 秒退避，并提供“继续等待”和“返回设备列表”。claim 已成功时不得让用户重复扫码。

### 6. 限制 WebView 外部 scheme

当前非 Relay origin 会全部交给系统打开。建议仅允许 `https` / `http` 外部链接，按需明确允许 `mailto`；直接拒绝 `file`、`javascript`、`data`、`intent` 及未知 scheme。Relay origin 比较必须包含 scheme、host 和有效端口。配对二维码的 Relay 比较也应加上 scheme，避免相同 host/port 下的协议降级。

### 7. 更新用户可见文案

将移动端中的 `dsh-remote` 统一改为 `dsh-mobile`。电脑端安装/启动指引使用：

```bash
dsh plugin --profile web add "/absolute/path/to/dsh-plugin"
dsh web
```

不要提示用户单独常驻运行 Companion，它已经跟随 `dsh web` 生命周期。

### 8. 补真实协议测试，不只测 Mock UI

建议新增以下自动化覆盖：

- `DioRelayService`：注册/登录、Bearer header、refresh token 轮换、并发 401 只 refresh 一次、refresh 失败退出。
- 配对：Relay scheme/host/port 不一致、claim 成功后延迟 confirm。
- WebView URL：精确生成 `/s/{deviceId}/?ticket=...`，不将 Bearer token 或 device secret 注入 WebView。
- WebView 状态：401 只换一次 ticket，503 重查设备，504 可重试，外部非安全 scheme 被拒绝。
- 构建门禁：release 默认 `useMock == false` 且 Relay 为 HTTPS。

## 不需要改动的部分

- `flutter_secure_storage` 保存 access/refresh token 的方向正确。
- `QueuedInterceptorsWrapper` + `_refreshInFlight` 合并并发 refresh 的方向正确。
- 手机仅持有用户 token 和一次性 web ticket，不持有 `deviceSecret` / `deviceToken`。
- InAppWebView 使用 HttpOnly Cookie 访问 DSH 的方向正确，不要改成把 access token 放入 URL 或 JavaScript。
- Android 生产默认禁止明文传输、仅对 loopback 开发地址例外的配置可以保留。

## 移动端交付标准

交付时请提供：

1. `flutter analyze` 无错误，`flutter test` 全部通过。
2. Android 和 iOS 真实 Relay 录屏或可复现测试记录。
3. 从扫码到 `session.prompt` 成功的请求时序，其中不含密码、token、Cookie 或任务正文。
4. 断网、DSH 停止、Companion 停止、Cookie 失效和 refresh token 失效五种状态的验收结果。

## 命名注意

电脑端本地包已统一为 `dsh-mobile`。但 npm 上无作用域的 `dsh-mobile` 已有现存包（检查时为 `0.1.0-alpha.6`）。面向公众发布前必须确认该包归属；如果不受本项目控制，应使用组织 scope，不得覆盖或依赖第三方同名包。

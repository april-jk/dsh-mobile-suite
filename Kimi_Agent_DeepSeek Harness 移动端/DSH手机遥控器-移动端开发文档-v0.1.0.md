# DSH 手机遥控器 — 移动端开发文档（v0.1.0 MVP）

> 本文档面向移动端开发者。
> 配套文档：《DSH 手机遥控器 — 插件端开发文档（v0.1.0）》，其中 §3 配对流程、§6.3 REST API、§6.4 转发规则与本文档强相关，开发前必读。
> 版本：0.1.0 ｜ 状态：设计定稿

---

## 1. 产品定位与 0.1 范围

用户在电脑上运行 DSH（DeepSeek Harness），通过本 App 随时随地访问自己电脑上的 DSH 界面。

**产品哲学：WebView 负责「看」，原生负责「管」。** 0.1 只做「看」+ 账号体系；「管」（推送、审批卡片）在 0.2 以原生层加入。因此 0.1 的 App 是一个克制的 WebView 壳，原生部分只做账号、设备管理和容器。

### 1.1 功能范围

**做（0.1.0）：**

1. 注册 / 登录 / 登录态维持（token 自动刷新）
2. 扫码配对绑定电脑设备（含手动输入配对码兜底）
3. 设备列表（在线状态、DSH 状态、重命名、解绑）
4. WebView 容器：完整加载远程 DSH Web UI 并正常交互
5. 异常状态呈现：设备离线 / DSH 未启动 / 加载失败重试

**不做（明确排除，勿提前实现）：**

- 推送通知、原生审批卡片（0.2，协议已在插件端预埋）
- 自建聊天/会话 UI
- 多账号切换、深色模式跟随之外的主题定制
- iPad / 平板横屏适配

---

## 2. 技术选型

- 框架：**Flutter 3.x**（一套代码出双端；WebView 能力与插件生态成熟）
  - 关键插件：`flutter_inappwebview`（必须用它而非官方 webview_flutter——需要拦截请求、注入脚本、处理 WS 升级）、`mobile_scanner`（扫码）、`flutter_secure_storage`（token 存储）、`dio`（REST）
  - 若团队全是原生背景，Swift(Kit) + Kotlin(Compose) 双原生亦可，本文档页面与接口设计不受影响；但不要混用 RN。
- 状态管理：Riverpod 或 Bloc，二选一，全项目统一。
- 最低系统：iOS 14 / Android 8.0。

---

## 3. 页面设计

共 5 个页面 + 1 个容器，全部竖屏。

### 3.1 登录/注册页 `LoginPage`

- 邮箱 + 密码（或邮箱验证码，与后端 §6.3 对齐，以其实现为准）。
- 登录成功 → 设备列表；若本地已有有效 token 则启动时跳过本页静默进入。
- 失败提示具体化：「邮箱或密码错误」「网络异常，请检查网络」，不要只弹「登录失败」。

### 3.2 设备列表页 `DeviceListPage`（首页）

- 每个设备卡片：设备名、状态点（绿=在线且 DSH 运行 / 黄=在线但 DSH 未启动 / 灰=离线）、最后在线时间。
- 数据来自 `GET /devices`，下拉刷新；进入页面前自动刷新一次。
- 卡片操作（左滑或长按）：重命名、解绑（二次确认：「解绑后需回到电脑前重新扫码」）。
- 离线/黄状态卡片仍可点击，进入容器页后由容器呈现对应空态（见 3.4），**不要**在列表层拦截禁止进入。
- 右上角「+」→ 扫码配对页。

### 3.3 扫码配对页 `PairPage`

- 默认相机扫码（二维码内容为 `{"v":1,"relay":"...","code":"482913"}`）。
- 底部入口「手动输入配对码」：6 位数字键盘，用于相机权限被拒或扫码失败的兜底。
- 流程：拿到 code → `POST /pair/claim` → 成功展示「绑定成功，设备：XXX」2s 后返回列表并刷新；失败原因具体化（码过期 / 码无效 / 频率受限）。
- **注意**：claim 成功后设备名由 Companion 在 confirm 阶段上报，App 需重新拉一次 `GET /devices` 再展示成功提示。

### 3.4 会话容器页 `SessionWebViewPage`（核心页面）

- 整页 WebView，顶栏只保留：返回按钮、设备名、一个「刷新」按钮。不要有地址栏。
- 加载地址：`https://relay.example.com/s/{deviceId}/`（鉴权见 §4.2）。
- 加载态骨架屏；错误态分三种，必须有明确文案与重试按钮：
  | 场景 | 判定 | 文案 |
  |---|---|---|
  | 设备离线 | 后端返回 503 + `{"reason":"device_offline"}` | 「电脑不在线。请确认电脑端的 dsh-remote 正在运行。」 |
  | DSH 未启动 | 503 + `{"reason":"dsh_offline"}` 或设备状态轮询为黄 | 「电脑在线，但 DSH 未启动。请在电脑上运行 npx @deepseek-ai/dsh web。」 |
  | 其他加载失败 | 超时 / 5xx / 断网 | 「连接失败，点击重试」 |
- WebView 内导航策略：`relay.example.com` 域内链接一律当前 WebView 打开；**域外链接一律跳系统浏览器**（防止 DSH 界面里的外链把用户带出后无法返回）。
- Android 物理返回键：WebView 可后退则后退，否则退出页面。

### 3.5 设置页 `SettingsPage`

- 账号信息、退出登录、版本号、意见反馈入口（mailto 或表单链接占位）。
- 0.1 不放通知设置（没有推送）。

---

## 4. 与云端的对接

### 4.1 REST（完整定义见插件端文档 §6.3，移动端用到以下子集）

| 用途 | 调用 |
|---|---|
| 登录 | `POST /auth/login` → `{accessToken, refreshToken}` |
| 刷新 | accessToken 过期（401）时 `POST /auth/refresh`，单飞模式（并发请求只刷新一次） |
| 设备列表 | `GET /devices`（Bearer accessToken） |
| 配对 | `POST /pair/claim {code}` |
| 重命名/解绑 | `PATCH` / `DELETE /devices/:id` |

### 4.2 WebView 的鉴权方式

WebView 无法方便地带 Bearer 头，采用**短期 ticket**：

1. 进入容器页前，App 调 `POST /web-ticket`（Bearer）→ 获得一次性 ticket（60s 有效，单次使用）。
2. WebView 加载 `https://relay.example.com/s/{deviceId}/?ticket=xxx`。
3. 云端校验 ticket 后种下 HttpOnly Cookie（作用域 `/s/{deviceId}`，有效期 2h），后续该 WebView 内所有请求（含 WS 升级）凭 Cookie 鉴权。
4. Cookie 过期导致页面 401 时，App 通过 URL 拦截识别，重新取 ticket 并 reload，**对用户无感**。

> `POST /web-ticket` 不在插件端文档 §6.3 表格中，属于移动端专用补充接口，需与后端确认排期；实现极轻（签一个一次性 JWT 即可）。

### 4.3 WebView 容器技术要点（逐条落实）

1. 使用 `flutter_inappwebview` 的 `InAppWebView`，开启：JavaScript、DOM Storage、Cookie 持久化、`useShouldOverrideUrlLoading`（域外跳浏览器）。
2. **不要拦截 WebSocket**——`flutter_inappwebview` 下 WS 由页面内核自行发出，云端按 Cookie 鉴权即可；拦截反而会断流式输出。
3. 注入能力预留：`onLoadStop` 时预留一个 `injectAdaptionCss()` 钩子，0.1 可为空实现，但架构上必须有——后续 DSH 桌面 UI 在窄屏上的适配补丁从这里下发（建议 0.2 改为从云端拉取补丁，免发版）。
4. 禁止 WebView 内打开文件选择器之外的任何系统弹窗；`window.open` 一律走域外规则。
5. 页面内长按弹出的系统菜单保留（用户需要复制文本），但禁用图片长按保存之外的扩展项无需处理，默认即可。

---

## 5. 状态与存储

- `accessToken` / `refreshToken`：存 `flutter_secure_storage`（Keychain / Keystore），**禁止**写 SharedPreferences/普通文件。
- 设备列表缓存：内存态即可，不持久化（每次进首页拉取）。
- 登录态：App 冷启动读 refreshToken → 尝试刷新 → 成功进首页，失败回登录页。
- 应用锁（指纹/人脸）：0.1 不做，0.2 与推送一起上。

---

## 6. 埋点（0.1 最小集）

只埋 6 个事件，用于验证核心假设「用户到底用手机端干什么」：

| 事件 | 触发 |
|---|---|
| `pair_success` | 配对成功 |
| `session_open` | 进入容器页 |
| `session_duration` | 退出容器页，带时长 |
| `device_offline_seen` | 看到设备离线空态 |
| `dsh_offline_seen` | 看到 DSH 未启动空态 |
| `webview_reload` | 手动点刷新 |

数据将直接决定 0.2/0.3 原生层做哪些页面，务必上。

---

## 7. 0.2 预告（不改架构，仅知悉）

0.2 将增加：推送通知（FCM/APNs）+ 原生审批卡片页。云端已预埋 `device_events`（权限请求/任务完成等结构化事件）。届时 App 新增：

- 推送到达 → 点开原生卡片：事件标题 + 「批准 / 拒绝 / 查看详情（跳 WebView）」。
- 卡片的「批准/拒绝」会调用新增的 `POST /events/:actionId/respond`，经云端 → Companion → 本地 DSH 完成操作。

0.1 的代码结构上请把「容器页」与「未来的原生功能页」保持平级路由，不要把所有逻辑堆进 WebView 页面控制器。

---

## 8. 0.1.0 验收标准

1. 新用户注册 → 登录 → 扫码绑定 → 列表出现设备，全程不超过 2 分钟。
2. 打开设备：DSH Web UI 完整加载，可浏览历史会话。
3. 在 WebView 内发送一条消息，流式输出正常渲染、不断流、不重复（WS 通道验证）。
4. 杀掉电脑端 DSH 进程：30s 内容器页显示「DSH 未启动」空态；重启 DSH 后点重试恢复正常。
5. 电脑休眠/断网 5 分钟恢复后：App 无需重新登录可继续使用。
6. 杀掉 App 重开：静默登录进首页，不弹登录页。
7. 解绑设备：列表移除，电脑端 Companion 提示已解绑；再次进入原链接显示设备离线文案。
8. WebView 内点击外部链接（如 github.com）：跳系统浏览器，App 内页面不丢失。
9. Android 返回键：页内可后退时后退，不可后退时退出容器页，不误退 App。
10. iOS/Android 双端安装包 ≤ 40MB。

### 测试要点

- 弱网：3G 节流下 WebView 加载与 WS 流式输出可用性。
- token 过期：手动把 accessToken 改废，触发静默刷新，WebView 不中断。
- 配对码：过期码、错误码、同一码被两台手机同时 claim。

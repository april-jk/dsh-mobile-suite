# Mobile WebView 性能整改要求

本文记录 2026-08-16 在生产 Relay 与 iOS Simulator WKWebView 上完成的实测结果，用于 Companion、Relay 与 DSH Web 静态资源链路整改。

## 已确认现象

- 生产 Relay：`https://dsh-relay-production.up.railway.app`
- 本机 DSH：`http://127.0.0.1:3080`
- 同一 HTML/CSS 经 Relay 前后的 SHA-256 完全一致，不存在静态内容损坏。
- 本机 DSH 首页响应约 2 ms。
- 生产 Relay 完整页面在 Chromium 中加载约 18.7 s，共 55 个资源，无失败请求。
- iOS WebView 增加缓存和字体修复后，`session_webview_ready` 首次实测约 12.1 s。
- `index-CSGf6Qzd.css` 为 67,798 bytes，经 Relay 约 0.8 s。
- `index-Dqw48FrP.js` 为 442,711 bytes，经 Relay 约 2.0 s。
- `vendor-Cjbwl5VI.js` 为 744,872 bytes，经 Relay 约 4.8 s。
- DSH 对 `Accept-Encoding: gzip` / `br` 仍返回原始大小，且静态资源没有可用的 `Cache-Control`。

## 根因

当前浏览器资源以 HTTP 请求进入 Relay，再封装为 JSON + Base64 通过单条 Companion WebSocket 转发。Base64 会额外增加约 33% 传输体积；DSH 静态资源不压缩且不可长期缓存，导致首次加载和重复进入都需要传输大量脚本与插件资源。

移动端只能显式启用 WebView 缓存，不能为跨进程隧道中的响应安全补写缓存或压缩语义。压缩必须在进入 Companion WebSocket 前完成，缓存头必须由 DSH 或 Companion/Relay 响应链路提供。

## P0 整改

1. 对文件名含内容哈希的 `/assets/*` 和稳定 revision 的 `/plugins/*` 响应增加：

   ```http
   Cache-Control: public, max-age=31536000, immutable
   ```

2. `/`、manifest 和无内容哈希入口不得 immutable，应使用 `no-cache` 或短缓存并支持重新验证。
3. 对 JavaScript、CSS、JSON、SVG、HTML 等可压缩类型启用 Brotli 或 gzip，并正确维护 `Content-Encoding`、`Vary: Accept-Encoding`，删除失效的 `Content-Length`。
4. SSE、WebSocket、已压缩字体/图片不得重复压缩或整包缓冲。
5. Companion 若执行压缩，应流式压缩 HTTP body，再按现有 `seq/final` 协议发送，不能等待完整响应后才返回首字节。

## P1 协议优化

- 评估 Tunnel Protocol v2 使用 WebSocket binary frame 传输 HTTP body，避免 JSON + Base64 的体积和 CPU 开销。
- 保留 JSON control frame；binary frame 必须包含 channel、sequence 和 final 元数据，继续支持多请求并发。
- 为 Relay 与 Companion 增加每请求 TTFB、总字节、压缩前后字节和总耗时指标，但不得记录 Cookie、token、任务正文或文件内容。

## 验收标准

- iOS 与 Android 第一次打开已配对电脑的 DSH 页面：`session_webview_ready < 5 s`。
- 同版本资源第二次打开：`session_webview_ready < 2 s`。
- 入口 HTML 更新后不得引用已被本机 DSH 删除的旧哈希资源。
- `/assets`、`/plugins`、`/api`、SSE 和 WebSocket 功能继续通过现有端到端测试。
- 本地与 Relay 解压后资源 SHA-256 必须一致。

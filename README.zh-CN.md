# DSH Mobile Suite

[English](README.md) | [简体中文](README.zh-CN.md)

用 Android 手机远程打开电脑上的 DeepSeek Harness Web UI，并通过原有界面创建和下达新任务。

> **非官方社区项目：** 本项目由社区独立开发和维护，未经 DeepSeek 审核、推荐或背书。当前 Companion、浏览器端与 Mobile 之间的 DSH 会话内容使用端到端加密。

DeepSeek Harness 社区展示帖：[Show Your Plugins! #2520](https://github.com/deepseek-ai/deepseek-harness/discussions/2520)（社区发现入口，不代表官方审核或背书）。

<table>
  <tr>
    <td><img src="docs/images/mobile-login.png" alt="DSH Mobile 登录与 Relay 选择" width="320"></td>
    <td><img src="docs/images/mobile-devices.png" alt="DSH Mobile 电脑列表" width="320"></td>
  </tr>
</table>

## 组成

本仓库以 Git submodule 锁定四个可独立开发和发布的开源组件：

| 目录 | 作用 | 独立仓库 |
| --- | --- | --- |
| `dsh-mobile/` | Flutter Android/iOS 客户端 | [april-jk/dsh-mobile](https://github.com/april-jk/dsh-mobile) |
| `dsh-plugin/` | 安装到 DSH 的电脑端插件与 Companion | [april-jk/dsh-mobile-plugin](https://github.com/april-jk/dsh-mobile-plugin) |
| `dsh-relay/` | 账号、配对、短期票据和流量中转服务 | [april-jk/dsh-relay](https://github.com/april-jk/dsh-relay) |
| `dsh-website/` | 宣传站、SEO 内容与 GitHub Pages 部署 | [april-jk/dsh-mobile-site](https://github.com/april-jk/dsh-mobile-site) |

手机不会直接连接电脑。电脑端插件只建立到 Relay 的出站 WSS 连接，DSH 继续监听 `127.0.0.1:3080`。

默认公共 Relay：`https://relay.dshmobile.online`

## 直接使用

在 [最新 Release](https://github.com/april-jk/dsh-mobile-suite/releases/latest) 下载：

- `dsh-mobile-android.apk`：已签名 Android 安装包
- `dsh-mobile-plugin.tgz`：预构建 DSH 插件包
- `dsh-relay-v*.tar.gz`：Relay 私有部署源码包
- `SHA256SUMS`：所有安装包的 SHA-256 校验值

先校验下载文件：

```bash
shasum -a 256 -c SHA256SUMS
```

直接从 GitHub 安装固定版本的插件并启动 DSH。该方式不要求全局安装 DSH、不需要克隆源码，也不需要填写本地文件路径：

```bash
npx @deepseek-ai/dsh plugin --profile web add "github:april-jk/dsh-mobile-plugin#v0.1.7"
npx @deepseek-ai/dsh web
```

在 Android 设备上安装 `dsh-mobile-android.apk`。首次安装第三方 APK 时，系统会要求允许浏览器或文件管理器安装未知应用。应用包名为 `io.github.apriljk.dshremote`。

## 配对和远程任务

1. 在手机应用注册或登录账号。
2. 在电脑的 DSH Web UI 打开 **Settings > Remote Access**，创建加密配对二维码。
3. 在手机电脑列表点 **+** 并扫码。由于六位码无法建立加密密钥，当前版本不支持仅输入六位码配对。
4. 选择在线电脑，手机会打开正常的 DSH Web UI。
5. 在 DSH 原有界面创建任务并提交指令。

### 增加浏览器访问端

电脑完成配对后，在 DSH 的 **Settings > Remote Access** 中点击 **生成浏览器访问码**。在另一台浏览器中打开链接，或使用该浏览器扫描二维码，然后登录同一个 Relay 账号。每个浏览器独立保存这台电脑的 E2EE 密钥，因此手机、iPhone Safari 和多个浏览器可以同时使用，不会互相解绑。

插件随 DSH Web 进程启停，无需另开后台进程。电脑离线、DSH 未启动或插件未连接 Relay 时，手机会显示该电脑离线。

## 私有部署 Relay

下载并解压 Release 中的 `dsh-relay-v*.tar.gz`，然后：

```bash
cp .env.example .env
# 编辑 .env，至少将 JWT_SECRET 换成长随机值
docker compose up -d --build
curl http://127.0.0.1:8787/health
```

公网使用时必须在 `8787` 端口前配置 HTTPS 反向代理，并为 `/data` 中的 SQLite 数据做持久化备份。MVP Relay 必须以单实例运行。

电脑和手机必须指向同一个 Relay：

```bash
DSH_RELAY=https://relay.example.com npx @deepseek-ai/dsh web
```

在手机登录页点 **Relay**，或登录后打开 **设置 > Relay 服务器**，输入同一个 HTTPS 地址。切换 Relay 会退出当前账号，因为不同 Relay 的账号和 Token 完全独立。

完整环境变量与资源限制见 [Relay README](https://github.com/april-jk/dsh-relay#readme)。

## 安全边界

- 公网传输必须使用 HTTPS/WSS；电脑端不会开放公网监听端口。
- 移动端只持有账号 Token 和短期 Web Ticket，不会得到电脑设备密钥。
- DSH HTTP、SSE 与 WebSocket 内容在 Mobile 和 Companion 之间端到端加密；Relay 只路由不透明密文帧，不持有内容密钥。
- Relay 仍可观察账号/设备关系、在线状态、连接时间、密文长度和流量时序。当前二维码预置密钥方案不提供前向保密。
- Relay 会保存账号、设备、配对和受限的访问日志元数据；公开商店发布前仍需完成账号删除、隐私政策和数据合规事项。

安全问题请按 [SECURITY.md](SECURITY.md) 私下报告，不要在公开 Issue 中披露可利用细节。

## 开发

```bash
git clone --recurse-submodules https://github.com/april-jk/dsh-mobile-suite.git
cd dsh-mobile-suite
git submodule update --init --recursive
```

组件代码在各自仓库提交。本仓库只维护跨端文档、统一 Release 工作流和经过验证的组件提交指针。贡献方式见 [CONTRIBUTING.md](CONTRIBUTING.md)。

每次向 `main` 推送或提交 Pull Request，都会执行各组件构建和测试、校验 Plugin 已提交 bundle、构建 Relay 容器与 Android APK，并检查公开网站。发布 Suite 时，先推送相同的组件 Tag、更新本仓库固定的 submodule，再向本仓库推送同名 Tag。Release 工作流会先确认 Tag 与 Plugin、Relay、Mobile 包版本完全一致，再生成已签名产物与校验和。

## 许可证

父仓库和四个组件均使用 [MIT License](LICENSE)。

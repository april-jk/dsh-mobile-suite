# DSH Mobile Suite

用 Android 手机远程打开电脑上的 DeepSeek Harness Web UI，并通过原有界面创建和下达新任务。

> **非官方社区项目：** 本项目由社区独立开发和维护，未经 DeepSeek 审核、推荐或背书。当前版本是 MVP，尚未实现端到端加密。

DeepSeek Harness 社区展示帖：[Show Your Plugins! #2520](https://github.com/deepseek-ai/deepseek-harness/discussions/2520)（社区发现入口，不代表官方审核或背书）。

<table>
  <tr>
    <td><img src="docs/images/mobile-login.png" alt="DSH Mobile 登录与 Relay 选择" width="320"></td>
    <td><img src="docs/images/mobile-devices.png" alt="DSH Mobile 电脑列表" width="320"></td>
  </tr>
</table>

## 组成

本仓库以 Git submodule 锁定三个可独立开发和发布的开源组件：

| 目录 | 作用 | 独立仓库 |
| --- | --- | --- |
| `dsh-mobile/` | Flutter Android/iOS 客户端 | [april-jk/dsh-mobile](https://github.com/april-jk/dsh-mobile) |
| `dsh-plugin/` | 安装到 DSH 的电脑端插件与 Companion | [april-jk/dsh-mobile-plugin](https://github.com/april-jk/dsh-mobile-plugin) |
| `dsh-relay/` | 账号、配对、短期票据和流量中转服务 | [april-jk/dsh-relay](https://github.com/april-jk/dsh-relay) |

手机不会直接连接电脑。电脑端插件只建立到 Relay 的出站 WSS 连接，DSH 继续监听 `127.0.0.1:3080`。

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

将插件包安装到 DSH Web profile，并启动 DSH：

```bash
dsh plugin --profile web add ./dsh-mobile-plugin.tgz
dsh web
```

也可以直接安装已锁定的公开插件版本：

```bash
dsh plugin --profile web add github:april-jk/dsh-mobile-plugin#v0.1.1
dsh web
```

在 Android 设备上安装 `dsh-mobile-android.apk`。首次安装第三方 APK 时，系统会要求允许浏览器或文件管理器安装未知应用。应用包名为 `io.github.apriljk.dshremote`。

## 配对和远程任务

1. 在手机应用注册或登录账号。
2. 在电脑的 DSH Web UI 打开 **Settings > Remote Access**，创建六位配对码或二维码。
3. 在手机电脑列表点 **+**，扫码或输入六位配对码。
4. 选择在线电脑，手机会打开正常的 DSH Web UI。
5. 在 DSH 原有界面创建任务并提交指令。

插件随 `dsh web` 启停，无需另开后台进程。电脑离线、DSH 未启动或插件未连接 Relay 时，手机会显示该电脑离线。

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
DSH_RELAY=https://relay.example.com dsh web
```

在手机登录页点 **Relay**，或登录后打开 **设置 > Relay 服务器**，输入同一个 HTTPS 地址。切换 Relay 会退出当前账号，因为不同 Relay 的账号和 Token 完全独立。

完整环境变量与资源限制见 [Relay README](https://github.com/april-jk/dsh-relay#readme)。

## 安全边界

- 公网传输必须使用 HTTPS/WSS；电脑端不会开放公网监听端口。
- 移动端只持有账号 Token 和短期 Web Ticket，不会得到电脑设备密钥。
- Relay 不持久化 DSH HTTP/WebSocket 请求体或响应体，但 MVP 没有应用层端到端加密，Relay 进程能够看到中转内容。
- Relay 会保存账号、设备、配对和受限的访问日志元数据；公开商店发布前仍需完成账号删除、隐私政策和数据合规事项。

安全问题请按 [SECURITY.md](SECURITY.md) 私下报告，不要在公开 Issue 中披露可利用细节。

## 开发

```bash
git clone --recurse-submodules https://github.com/april-jk/dsh-mobile-suite.git
cd dsh-mobile-suite
git submodule update --init --recursive
```

组件代码在各自仓库提交。本仓库只维护跨端文档、统一 Release 工作流和经过验证的组件提交指针。贡献方式见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 许可证

父仓库和三个组件均使用 [MIT License](LICENSE)。

# DSH Remote Public Contracts

This directory is the integration source of truth for the three independently versioned applications.

| Document | Consumer | Purpose |
| --- | --- | --- |
| [API-v1.md](API-v1.md) | Mobile, Companion, Relay | Account, pairing, device, ticket, and WebView HTTP contracts |
| [Tunnel-Protocol-v1.md](Tunnel-Protocol-v1.md) | Companion, Relay | WSS authentication, multiplexed HTTP and WebSocket forwarding |
| [Integration-Plan.md](Integration-Plan.md) | All teams | Local and Railway test sequence, ownership, and release gates |
| [DSH-远程MVP-跨端接口.md](DSH-远程MVP-跨端接口.md) | All teams | Concise implementation-aligned contract summary |
| [移动端-MVP-修改建议.md](移动端-MVP-修改建议.md) | Mobile | Review findings and acceptance criteria for connecting the Flutter app to the deployed MVP |
| [WebUI-远程访问管理方案.md](WebUI-远程访问管理方案.md) | Plugin, Relay, Mobile | Move pairing into DSH WebUI and add authorized-device and access-session management |
| [Mobile-WebView-Performance.md](Mobile-WebView-Performance.md) | Companion, Relay, Mobile | Measured tunnel bottlenecks, compression/cache requirements, and WebView performance gates |

The product baseline remains in `../Kimi_Agent_DeepSeek Harness 移动端/`. Where an implementation detail was not specified there, the API and protocol documents in this directory make the MVP decision explicit. When examples differ, `API-v1.md` and `Tunnel-Protocol-v1.md` are authoritative.

## Install the computer plugin

From a local checkout:

```bash
cd dsh-plugin
npm install
npm run build
dsh plugin --profile web add "/absolute/path/to/dsh-plugin"
dsh web
```

Profiles that previously installed the development name must first run `dsh plugin --profile web remove dsh-mobile-remote-companion`, then add the local path again. This prevents both bundle identities from starting together.

The local installable package name is `dsh-mobile`. Its DSH bundle starts and stops the Companion with `dsh web`, pins workspace selection to DSH's in-browser directory picker, and stores pairing credentials at `~/.dsh-remote/config.json` by default. The unscoped npm name is already occupied, so public distribution must first resolve package ownership or use an organization scope.

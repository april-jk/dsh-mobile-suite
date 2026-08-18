# DSH Remote Public Contracts

This directory is the integration source of truth for the three independently versioned applications.

| Document | Consumer | Purpose |
| --- | --- | --- |
| [API-v1.md](API-v1.md) | Mobile, Companion, Relay | Account, pairing, device, ticket, and WebView HTTP contracts |
| [Tunnel-Protocol-v1.md](Tunnel-Protocol-v1.md) | Companion, Relay | WSS authentication, multiplexed HTTP and WebSocket forwarding |
| [Tunnel-E2EE-v1.md](Tunnel-E2EE-v1.md) | Mobile, Companion, Relay | QR trust bootstrap, sealed handshake, AEAD framing, and downgrade policy |
| [Browser-Remote-and-Relay-Admin-v1.md](Browser-Remote-and-Relay-Admin-v1.md) | Browser, Companion, Relay | Safari access, browser E2EE gateway, and isolated Relay statistics admin |
| [Integration-Plan.md](Integration-Plan.md) | All teams | Local and Railway test sequence, ownership, and release gates |
| [DSH-远程MVP-跨端接口.md](DSH-远程MVP-跨端接口.md) | All teams | Concise implementation-aligned contract summary |
| [移动端-MVP-修改建议.md](移动端-MVP-修改建议.md) | Mobile | Review findings and acceptance criteria for connecting the Flutter app to the deployed MVP |
| [WebUI-远程访问管理方案.md](WebUI-远程访问管理方案.md) | Plugin, Relay, Mobile | Move pairing into DSH WebUI and add authorized-device and access-session management |
| [Mobile-WebView-Performance.md](Mobile-WebView-Performance.md) | Companion, Relay, Mobile | Measured tunnel bottlenecks, compression/cache requirements, and WebView performance gates |

The product baseline remains in `../Kimi_Agent_DeepSeek Harness 移动端/`. Where an implementation detail was not specified there, the API and protocol documents in this directory make the MVP decision explicit. When examples differ, `API-v1.md` and `Tunnel-Protocol-v1.md` are authoritative.

## Install the computer plugin

From the immutable GitHub release tag:

```bash
dsh plugin --profile web add github:april-jk/dsh-mobile-plugin#v0.1.7
dsh web
```

For local development, build `dsh-plugin` and add its absolute directory path instead. Profiles that previously installed the development name must first run `dsh plugin --profile web remove dsh-mobile-remote-companion`, then add the current package. This prevents both bundle identities from starting together.

The installable package name is `@april-jk/dsh-mobile`; its stable Cordis ID and CLI command remain `dsh-mobile`. Its DSH bundle starts and stops the Companion with `dsh web`, pins workspace selection to DSH's in-browser directory picker, and stores pairing credentials at `~/.dsh-remote/config.json` by default. The project is unofficial and independently maintained by the community.

After a computer is paired, the Companion's DSH settings can generate a browser-access QR/link. Each browser stores the computer's E2EE key independently, so a phone and multiple browsers can use the same computer concurrently. Browser enrollment does not create or replace a Relay computer device.

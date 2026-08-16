# DSH Remote Public Contracts

This directory is the integration source of truth for the three independently versioned applications.

| Document | Consumer | Purpose |
| --- | --- | --- |
| [API-v1.md](API-v1.md) | Mobile, Companion, Relay | Account, pairing, device, ticket, and WebView HTTP contracts |
| [Tunnel-Protocol-v1.md](Tunnel-Protocol-v1.md) | Companion, Relay | WSS authentication, multiplexed HTTP and WebSocket forwarding |
| [Integration-Plan.md](Integration-Plan.md) | All teams | Local and Railway test sequence, ownership, and release gates |
| [DSH-远程MVP-跨端接口.md](DSH-远程MVP-跨端接口.md) | All teams | Concise implementation-aligned contract summary |

The product baseline remains in `../Kimi_Agent_DeepSeek Harness 移动端/`. Where an implementation detail was not specified there, the API and protocol documents in this directory make the MVP decision explicit. When examples differ, `API-v1.md` and `Tunnel-Protocol-v1.md` are authoritative.

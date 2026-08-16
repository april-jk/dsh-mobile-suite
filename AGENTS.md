# DSH Remote Workspace

This root repository coordinates four independently versioned applications. It owns shared documentation and planning artifacts only; do not add application source code at the root.

## Directory ownership

| Directory | Owner and responsibility | Git repository |
| --- | --- | --- |
| `dsh-mobile/` | Flutter mobile client: account UI, pairing scanner, device list, WebView container | Independent |
| `dsh-plugin/` | Node.js Companion CLI on the user's computer: pairing, DSH discovery, Relay connection, local HTTP/WS forwarding | Independent |
| `dsh-relay/` | Cloud Relay: accounts, pairing, devices, tickets, WSS routing, persistence, deployment | Independent |
| `dsh-website/` | Public SEO website: product pages, technical content, downloads, and GitHub Pages deployment | Independent |
| `dsh-公共文档/` | Cross-team API, protocol, error, release, and integration contracts | Root |
| `Kimi_Agent_DeepSeek Harness 移动端/` | Product source documents; treat as the v0.1 baseline | Root |

Do not modify another application's source directory without the owning task's explicit scope. Cross-component behavior changes start with a proposal and versioned update under `dsh-公共文档/`.

## MVP objective

The first release proves a single user flow: a phone pairs with a computer running DSH, opens the remote DSH Web UI, and submits a new task through the normal DSH interface. The computer's DSH listener stays on `127.0.0.1:3080`; all remote connectivity is outbound through the Relay.

Deferred from MVP: end-to-end encryption, push notifications, native approvals, email verification/recovery, reliable DSH event extraction, and multi-instance Relay coordination.

## Cross-team contract rules

1. REST, WebView session, and Companion tunnel contracts are defined in `dsh-公共文档/`; implementation-specific types must conform to them.
2. Use version `v: 1` for every tunnel envelope. Additive payload fields are allowed; renaming/removing fields requires a new version and migration notes.
3. The mobile client never receives a device token or device secret. It uses account access tokens and short-lived web tickets only.
4. The Relay stores hashes of passwords, refresh tokens, and device tokens. It never logs or persists HTTP/WS tunnel payloads.
5. The Companion never exposes DSH publicly and must rewrite the upstream `Host` header to `127.0.0.1:3080`.
6. Any error a mobile user can see must use a documented status and `reason` code.

## Development and verification

- Node.js 18+ is required for the Companion and Relay.
- Test locally before cloud deployment. The Relay must be configurable through environment variables and deployable to Railway.
- Keep public API tests in the Relay, protocol compatibility tests in the owning repository, and end-to-end smoke instructions in `dsh-公共文档/`.
- Never commit real credentials, Railway tokens, generated SQLite databases, or `.env` files.
- Before finishing work in any repository, run its available format, typecheck, tests, and relevant integration checks.

## Git and commits

- Commit root documentation changes in the root repository.
- Commit each application change from within that application's own repository; never stage nested repository contents from the root.
- Every commit follows the Lore format: an intent-first subject followed by useful trailers such as `Constraint`, `Confidence`, `Scope-risk`, `Tested`, and `Not-tested`.

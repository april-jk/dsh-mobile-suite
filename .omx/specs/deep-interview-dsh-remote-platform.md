# DSH Remote Platform Execution Specification

## Metadata

- Profile: standard
- Rounds: 5
- Final ambiguity: 0.12
- Threshold: 0.20
- Context: greenfield
- Context snapshot: `.omx/context/dsh-remote-platform-20260816T052020Z.md`

## Intent

Prove that DeepSeek Harness can be used remotely from a phone without exposing its localhost listener, then deploy the Relay publicly for MVP users.

## Desired outcome

A locally reproducible and Railway-deployable system in which a Companion running on the user's computer pairs with an account, maintains an outbound WSS connection, and forwards authorized mobile WebView HTTP and WebSocket traffic to `127.0.0.1:3080`.

## In scope

- Independent TypeScript projects in `dsh-plugin/` and `dsh-relay/`.
- A versioned, shared protocol contract distributed without coupling the two Git repositories to a monorepo.
- Companion CLI commands: `start`, `pair`, `status`, `unpair`; clean shutdown covers the documented `stop` intent for MVP.
- Secure local credential/config persistence and reconnecting WSS client.
- DSH health probing and online/offline status reporting.
- HTTP and WebSocket tunnel forwarding.
- Relay email/password registration, login, refresh-token rotation, pairing, device listing/rename/unbind, web tickets, cookie sessions, device WSS gateway, and tunnel routing.
- SQLite persistence suitable for a single Railway instance, with an explicit volume path.
- Local end-to-end and integration tests.
- Railway deployment configuration and operator instructions.
- Cross-team REST, WebView, pairing, and tunnel protocol documentation under `dsh-公共文档/`.
- Event envelope, Relay storage capped at 50 events per device, and Companion collection extension point.

## Out of scope

- Mobile app implementation or modification.
- End-to-end encryption.
- Push notifications and approval cards.
- Email verification, password recovery, and social login.
- Heuristic parsing of DSH WebSocket content for events.
- Guaranteed support for HTTP response bodies larger than 1 MiB.
- Guaranteed SSE long-stream support.
- Multi-instance Relay coordination for the first public MVP.

## Decision boundaries

The implementation agent may choose package tooling, TypeScript libraries, schema details, token lifetimes consistent with the product documents, test structure, and Railway configuration without further approval. It may not broaden the security claims, modify the mobile project, add E2E encryption, or silently change externally documented API/protocol contracts.

## Constraints

- Node.js 18 or newer.
- Companion has no native runtime dependencies.
- TLS/WSS required outside local development.
- Tunnel payload bodies must never be logged or persisted.
- Passwords, refresh tokens, device tokens, and web tickets are stored only as hashes where server-side persistence is required.
- Pairing is six digits, expires in five minutes, and is single use.
- Existing v0.1.0 design documents remain authoritative unless this spec explicitly narrows them.
- Each implementation directory has its own Git history and must receive Lore-format commits.

## Acceptance criteria

1. A clean local setup starts Relay and a fake or real DSH endpoint with documented commands.
2. Companion creates a pairing code and can be claimed by an authenticated user.
3. Companion persists its device credential and reconnects without re-pairing.
4. `GET /devices` reflects Companion and DSH online state.
5. A web ticket establishes an HttpOnly device-scoped cookie.
6. Authorized `/s/:deviceId/*` HTTP requests reach local DSH and return their response.
7. Authorized WebSocket traffic is bidirectionally forwarded so a new DSH task can be submitted and streamed results can return.
8. An offline Companion returns `503 {"reason":"device_offline"}`; an online Companion with offline DSH returns `503 {"reason":"dsh_offline"}`.
9. Unbinding immediately invalidates the device token and disconnects or rejects the Companion.
10. Tests cover authentication, pairing, authorization isolation, web-ticket single use, HTTP tunnel, WebSocket tunnel, reconnect behavior, and payload-log redaction.
11. Relay deploys on Railway with persistent SQLite storage and a successful cloud smoke test.
12. Public API and protocol documentation is sufficient for the mobile agent to integrate independently.

## Assumptions and resolutions

- Public MVP can initially be single-instance: accepted by choosing SQLite plus one persistent Railway volume.
- TLS-only content confidentiality is acceptable for MVP: explicitly accepted, must be disclosed.
- Event extraction must not delay remote interaction: accepted; only scaffolding is required.
- Real DSH interface details may vary: tunnel is designed as transparent HTTP/WS forwarding and tests use a protocol-faithful fixture.

## Technical evidence

- DSH is documented to listen on `127.0.0.1:3080` and forbid public host binding.
- The mobile design requires `/web-ticket`, `/s/{deviceId}/`, device-state reasons, and Cookie-authenticated WebSocket upgrades.
- The plugin design specifies a single versioned WSS connection with channel-based multiplexing.

# DSH Remote Platform Context

## Task statement

Design and implement the plugin-side Companion CLI and the cloud Relay service for the DSH mobile remote-control product.

## Desired outcome

A new, Git-tracked project that implements the complete path from a mobile WebView request, through the Relay, to the user's locally running DeepSeek Harness instance.

## Stated solution

- Node.js Companion CLI installed and run through `npx`.
- Cloud Relay providing authentication, pairing, device management, WSS gateway, and HTTP/WebSocket tunneling.
- Shared versioned protocol types between Companion and Relay.
- Clarify material product and deployment choices before implementation.

## Probable intent hypothesis

Deliver an MVP that can later be connected to the separately developed Flutter client, while preserving a clean protocol upgrade path for push approvals and end-to-end encryption.

## Known facts and evidence

- The workspace contains finalized v0.1.0 mobile and plugin-side design documents.
- The workspace contained no implementation and was not a Git repository.
- Git tracking has now been initialized at the workspace root.
- DSH is expected to listen on `127.0.0.1:3080`.
- v0.1 requires pairing, device status, HTTP tunneling, WebSocket tunneling, and event-channel scaffolding.

## Constraints

- Companion requires Node.js 18 or newer and must avoid native dependencies.
- Cloud traffic must use TLS/WSS in deployment.
- Tunnel payload contents must not be persisted or logged.
- Device credentials must be protected at rest.
- v0.1 does not provide end-to-end encryption and must retain a protocol upgrade path.
- Existing design documents are the current product baseline unless explicitly revised during the interview.

## Unknowns and open questions

- Primary intended use: local development proof, private beta, or production deployment.
- Hosting target, public domain, TLS termination, and operational environment.
- Authentication method and email delivery requirements.
- Database choice and expected scale.
- Monorepo tooling, package name, and publishing expectations.
- Whether a mobile client or external API contract is already being developed elsewhere.
- Definition of completion for this implementation phase.

## Decision-boundary unknowns

- Which architectural and product decisions may be made autonomously.
- Which security, hosting, and compatibility choices require explicit approval.
- Whether accepted v0.1 limitations remain acceptable.

## Likely codebase touchpoints

- `packages/protocol`
- `packages/companion`
- `packages/relay`
- integration tests and local end-to-end test fixtures
- deployment and operator documentation

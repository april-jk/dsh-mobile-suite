# DSH Remote Platform Interview Summary

## Outcome

Implement only the computer-side Companion CLI and cloud Relay. A separate agent owns the mobile app. The first milestone is a working MVP that lets a phone open the remote DSH Web UI and submit new task instructions.

## Decisions

1. `dsh-plugin/` and `dsh-relay/` are independent Git repositories.
2. Local end-to-end validation comes first, followed by Railway deployment and a public-cloud validation pass.
3. Public MVP authentication uses email and password. Passwords are securely hashed. Email verification, password reset, and social login are deferred.
4. End-to-end encryption is deferred. MVP security relies on TLS/WSS, strict authorization, hashed credentials, and a prohibition on tunnel payload logging.
5. Event protocol and Relay persistence scaffolding are included, but DSH event inference is not. Companion performs capability detection and exposes an extension point.
6. The mobile-facing API and tunnel protocol are documented under `dsh-公共文档/`.

## Scope boundary

The MVP must support registration/login/refresh, pairing, device management and status, Web tickets, HTTP tunneling, WebSocket tunneling, reconnect behavior, local reproducible testing, Railway deployment configuration, and cross-component API documentation.

Deferred: mobile implementation, push notifications, native approval cards, complete event extraction, end-to-end encryption, email delivery, password reset, responses larger than the documented MVP limit, and guaranteed SSE streaming.

## Pressure pass

The initial goal combined a public MVP with a Relay capable of decrypting traffic. The security tradeoff was revisited explicitly. The user accepted the v0.1 TLS-only model to prioritize proving the remote-use path, with public disclosure of the limitation.

## Transcript

- User: Build the plugin and Relay, design the chain, initialize Git, and clarify uncertainties before coding.
- Assistant: Asked whether the target was a local prototype, private beta, or public MVP.
- User: Deliver an MVP; mobile is owned separately; use distinct plugin and Relay directories and a shared documentation directory.
- Assistant: Asked for the initial runtime and deployment target.
- User: Test locally, then deploy on Railway for a public MVP.
- Assistant: Proposed email/password authentication without email verification or password recovery.
- User: Accepted.
- Assistant: Challenged the TLS-only security model for a public MVP.
- User: Accepted deferring end-to-end encryption.
- Assistant: Proposed protocol/storage scaffolding for events without unreliable WebSocket content inference.
- User: Accepted; prioritize remote DSH use and submitting new task instructions.

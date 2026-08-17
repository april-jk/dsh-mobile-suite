# Integration and Release Plan

## Ownership

| Milestone | Mobile | Plugin | Relay |
| --- | --- | --- | --- |
| Account and token lifecycle | Integrate | - | Implement |
| Pair QR/claim | Integrate scanner | Create/confirm | Implement |
| Device list and statuses | Integrate UI | Report status | Implement |
| WebView ticket and cookie | Integrate container | - | Implement |
| Remote DSH HTTP/WS | Validate UI | Forward local DSH | Route tunnel |
| Railway deployment | Validate cloud URL | Set Relay URL | Deploy and operate |

## Local end-to-end sequence

1. Build `dsh-plugin` 0.1.4, then install it with `dsh plugin --profile web add "/absolute/path/to/dsh-plugin"`.
2. Start Relay with a development database and an explicit local JWT secret.
3. Start `dsh web`; the installed bundle starts Companion and binds it to DSH's actual web port. Open **Settings > Remote Access** and explicitly generate the QR payload/code when unpaired.
4. Register/login in mobile, claim the code, and wait for Companion confirmation.
5. Verify `/devices` reports `online` plus `dshStatus: online`.
6. Request a ticket, establish `/client-tunnel`, load the Mobile loopback origin, and submit a new DSH task.
7. Verify HTTP, SSE, and WebSocket traffic is sealed end to end and Relay observations contain no canary plaintext.
8. Stop local DSH and confirm `503 dsh_offline`; stop Companion and confirm `503 device_offline`.
9. Unbind from mobile and verify Companion cannot reconnect with its revoked token.

The MVP cloud verification on 2026-08-16 completed steps 1-7 against `https://dsh-relay-production.up.railway.app`: the remote browser loaded DSH assets and client plugins, selected a host directory through `host.listDirectory`, created a workspace/session, and delivered `session.prompt`. Model execution then stopped at the expected `MISSING_CREDENTIAL` boundary because the isolated DSH profile intentionally had no DeepSeek API key.

## Railway release gate

- Set `JWT_SECRET` to a generated production secret; never use the development default.
- Configure persistent storage for SQLite, or replace the Store implementation before scaling beyond one instance.
- Railway must provide HTTPS before allowing mobile WebView use; configure the public Relay origin in the Companion and mobile build.
- Run the complete local sequence against the Railway URL with a real computer-side Companion.
- Confirm application logs contain request metadata only and no authorization header, password, token, HTTP body, or WebSocket frame content.
- Reject QR v1, six-digit-only pairing, missing keys, missing capability, replay, reordered frames, and all plaintext fallback attempts.
- Verify ciphertext/tag/AAD mutation closes the secure session before content reaches DSH.
- Verify oversized API, tunnel HTTP, and WebSocket inputs are rejected without restarting the Relay.
- Verify per-client rate limits and per-device tunnel concurrency limits return documented `413`/`429` errors.

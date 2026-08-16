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

1. Start local DSH on `127.0.0.1:3080`.
2. Start Relay with a development database and an explicit local JWT secret.
3. Start Companion pointed to the local Relay; run pairing to produce a QR payload and code.
4. Register/login in mobile, claim the code, and wait for Companion confirmation.
5. Verify `/devices` reports `online` plus `dshStatus: online`.
6. Request a ticket, load `/s/:deviceId/`, and submit a new DSH task.
7. Verify browser-to-Companion WebSocket traffic transports streaming output.
8. Stop local DSH and confirm `503 dsh_offline`; stop Companion and confirm `503 device_offline`.
9. Unbind from mobile and verify Companion cannot reconnect with its revoked token.

## Railway release gate

- Set `JWT_SECRET` to a generated production secret; never use the development default.
- Configure persistent storage for SQLite, or replace the Store implementation before scaling beyond one instance.
- Railway must provide HTTPS before allowing mobile WebView use; configure the public Relay origin in the Companion and mobile build.
- Run the complete local sequence against the Railway URL with a real computer-side Companion.
- Confirm application logs contain request metadata only and no authorization header, password, token, HTTP body, or WebSocket frame content.
- Document the TLS-only MVP security limitation before opening public access.

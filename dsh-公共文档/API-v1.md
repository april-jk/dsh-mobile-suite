# DSH Remote API v1

Base URL: `https://<relay-host>` in cloud environments and `http://127.0.0.1:<port>` locally. JSON requests and responses use `application/json` unless noted.

## Authentication

Access tokens are bearer tokens with a seven-day lifetime. Refresh tokens are opaque, single-use credentials with a 30-day lifetime. The mobile client stores both in secure storage. The Relay stores only a refresh-token hash.

REST API errors use `{"error":"<code>"}`. Errors returned by the WebView proxy use `{"reason":"<code>"}` because they are rendered as user-facing connection states by the mobile client.

Every JSON API body is limited to 64 KiB by default. Oversized bodies return `413 {"error":"request_too_large","limit":65536}`. Rate-limited API calls return `429 {"error":"rate_limited","retryAfter":<seconds>}` with the same delay in the `Retry-After` header. Authentication and pairing routes have stricter per-client limits in addition to the general API limit.

## Mobile version policy

### `GET /app/version?platform=android|ios` (Public)

Returns the release policy for one mobile platform:

```json
{
  "platform": "android",
  "latestVersion": "0.2.0",
  "minimumVersion": "0.1.3",
  "downloadUrl": "https://play.google.com/store/apps/details?id=io.github.apriljk.dshremote",
  "releaseNotes": "Improved remote session stability."
}
```

`latestVersion` produces a dismissible update prompt when it is newer than the installed version. `minimumVersion` produces a non-dismissible prompt when it is newer than the installed version. Versions use numeric dotted ordering such as `1.4.2`; prerelease versions sort below the corresponding stable version. `downloadUrl` and `releaseNotes` may be `null` when no update is published. Unknown platforms return `400 {"error":"unsupported_platform"}`.

The mobile client checks after launch, whenever it returns to the foreground, and when the user manually checks from Settings. The Relay may cache this public response for at most five minutes. Operators must publish the store/download target before increasing either configured version.

### `POST /auth/register`

Request:

```json
{"email":"person@example.com","password":"at-least-8-characters"}
```

Returns `201` with the same body as login. Reject duplicate emails with `409 {"error":"email_exists"}`. Passwords must have a minimum length of 8 characters for MVP.

### `POST /auth/login`

Request: same fields as registration. Returns `200`:

```json
{"accessToken":"...","refreshToken":"..."}
```

Reject invalid credentials with `401 {"error":"invalid_credentials"}`. Do not reveal whether an email exists.

### `POST /auth/refresh`

Request: `{"refreshToken":"..."}`. Returns a rotated token pair. A used, expired, or invalid refresh token returns `401 {"error":"invalid_refresh_token"}`.

All routes below marked **User** require `Authorization: Bearer <accessToken>`.

## Pairing

The six-digit code and device secret are deliberately separate. The Companion owns the secret; the mobile client receives only the device identity after claim.

### `POST /pair/session` (Companion)

Rate-limited, no user authentication. Returns:

```json
{
  "deviceId":"dev_xxx",
  "deviceSecret":"hex-32-bytes",
  "code":"482913",
  "expiresAt":1786860000000
}
```

`expiresAt` is Unix time in milliseconds. The Companion also generates a local 32-byte E2EE master key and presents `{"v":2,"relay":"https://<relay-host>","code":"482913","e2eeKey":"<base64url>"}`. The key is never sent to the Relay. Production Mobile accepts only this QR form; six-digit manual pairing cannot establish the E2EE trust root.

### `POST /pair/claim` (User)

Request: `{"code":"482913"}`. Returns `200 {"deviceId":"dev_xxx"}`. Invalid, expired, used, or another user's claimed code returns `409 {"error":"invalid_or_expired_code"}`.

### `POST /pair/confirm` (Companion)

Request:

```json
{"deviceId":"dev_xxx","deviceSecret":"...","deviceName":"Watson's MacBook Air"}
```

Before claim it returns `202 {"status":"pending"}`. On success it returns `200 {"deviceToken":"..."}`. The Companion persists this token and authenticates its WSS connection using it. The Relay stores only its hash.

## Devices

### `GET /devices` (User)

Returns `200`:

```json
{
  "devices":[{
    "id":"dev_xxx",
    "name":"Watson's MacBook Air",
    "online":true,
    "dshStatus":"online",
    "lastSeenAt":1786860000000
  }]
}
```

`online` reflects the device WSS connection. `dshStatus` is the Companion's latest local health observation. `lastSeenAt` is Unix time in milliseconds or `null`.

### `PATCH /devices/:id` (User)

Request: `{"name":"New name"}`. Returns `200 {"ok":true}`. The caller must own the device. Mobile refreshes `GET /devices` after success.

### `DELETE /devices/:id` (User)

Returns `200 {"ok":true}`. The Relay immediately revokes the device token and closes the active Companion connection. A later Companion reconnect must receive `auth_fail` and return to pairing.

### `POST /device-management/:id/unbind` (Companion)

Requires `Authorization: Device <deviceToken>`. Returns `200 {"ok":true}` after atomically removing the account ownership and revoking the device token. The Relay closes the active Companion connection and ends active WebView access sessions with reason `unbound`.

This endpoint is used by the local DSH **Remote Access** settings page and the `dsh-mobile unpair` recovery command. An invalid or already revoked device credential returns `401 {"error":"invalid_device_token"}`. The Companion clears its local `deviceId`, `deviceSecret`, and `deviceToken` only after the Relay acknowledges the unbind, then returns to the unpaired state. Remote WebView requests must not be allowed to invoke this operation.

## WebView authorization and proxy

### `POST /web-ticket` (User)

Request: `{"deviceId":"dev_xxx"}`. For a connected `sealed-tunnel-v1` Companion it returns `200 {"ticket":"...","expiresIn":60,"accessSessionId":"access_xxx","tunnelUrl":"wss://<relay-host>/client-tunnel","e2eeRequired":true}`. Tickets are single-use and bound to the account, device, and access session. A missing secure capability returns `426 {"reason":"e2ee_required"}`.

The mobile client connects with the ticket in an `Authorization: WebTicket` header, then loads its private loopback gateway:

```
http://127.0.0.1:<ephemeral>/?bootstrap=<local-capability>
```

The loopback gateway consumes the local bootstrap capability, sets an `HttpOnly; SameSite=Strict; Path=/` cookie, and serves decrypted responses. DSH absolute `/assets`, `/plugins`, `/api`, and root WebSocket URLs remain on the loopback origin. No Relay bearer token, device credential, or E2EE key enters WebView JavaScript.

Relay `/client-tunnel` forwards only opaque E2EE frames. The legacy `/s/:deviceId` proxy is disabled by default and MUST NOT be used as an automatic fallback.

## User-facing proxy errors

| HTTP | Body | Mobile meaning |
| --- | --- | --- |
| `401` | `{"reason":"invalid_local_session"}` | Recreate the local proxy and request a new ticket |
| `403` | `{"reason":"invalid_local_origin"}` | Reject a loopback request with an unexpected Host |
| `403` | `{"error":"forbidden"}` | The account does not own the device or cannot request a ticket |
| `413` | `{"reason":"request_too_large"}` | The forwarded request exceeds the Mobile inner limit |
| `429` | `{"reason":"too_many_tunnels"}` | Mobile or Companion inner-channel capacity is temporarily exhausted |
| `429` | `{"error":"rate_limited","retryAfter":<seconds>}` | The client exceeded a Relay API or secure-frame rate limit |
| `502` | `{"reason":"secure_tunnel_failed"}` | The authenticated tunnel failed or rejected an inner response |
| `503` | `{"reason":"device_offline"}` | Computer/Companion is offline |
| `503` | `{"reason":"dsh_offline"}` | Computer is online but local DSH is unavailable |
| `504` | `{"reason":"tunnel_timeout"}` | Retryable connectivity failure |

Mobile and Companion default inner boundaries are 2 MiB per request, 32 MiB per response, 32 concurrent HTTP channels, and 16 concurrent WebSocket channels per secure session. Relay separately limits secure sessions, frame rate, ciphertext bytes, handshake time, and idle time. Deployments may lower these limits. Clients must treat `429` and `503` capacity responses as retryable with backoff.

The mobile client must allow WebView navigation inside the active loopback origin only. External HTTPS origins open in the operating system browser; all other origins are blocked.

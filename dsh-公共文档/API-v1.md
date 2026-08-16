# DSH Remote API v1

Base URL: `https://<relay-host>` in cloud environments and `http://127.0.0.1:<port>` locally. JSON requests and responses use `application/json` unless noted.

## Authentication

Access tokens are bearer tokens with a seven-day lifetime. Refresh tokens are opaque, single-use credentials with a 30-day lifetime. The mobile client stores both in secure storage. The Relay stores only a refresh-token hash.

### `POST /auth/register`

Request:

```json
{"email":"person@example.com","password":"at-least-12-characters"}
```

Returns `201` with the same body as login. Reject duplicate emails with `409 {"reason":"email_taken"}`. Password complexity policy is documented by Relay implementation, but must have a minimum length of 12.

### `POST /auth/login`

Request: same fields as registration. Returns `200`:

```json
{"accessToken":"...","refreshToken":"...","expiresIn":604800}
```

Reject invalid credentials with `401 {"reason":"invalid_credentials"}`. Do not reveal whether an email exists.

### `POST /auth/refresh`

Request: `{"refreshToken":"..."}`. Returns a rotated token pair. A used, expired, or invalid refresh token returns `401 {"reason":"invalid_refresh_token"}`.

All routes below marked **User** require `Authorization: Bearer <accessToken>`.

## Pairing

The six-digit code and device secret are deliberately separate. The Companion owns the secret; the mobile client receives only the device identity after claim.

### `POST /pair/session` (Companion)

Rate-limited, no user authentication. Returns:

```json
{
  "deviceId":"dev_xxx",
  "deviceSecret":"hex-32-bytes",
  "pairCode":"482913",
  "expiresAt":"2026-08-16T06:00:00.000Z",
  "relay":"wss://<relay-host>/device"
}
```

The Companion presents the QR payload `{"v":1,"relay":"wss://<relay-host>","code":"482913"}` and retains `deviceId` plus `deviceSecret` locally with restrictive file permissions.

### `POST /pair/claim` (User)

Request: `{"code":"482913"}`. Returns `202 {"deviceId":"dev_xxx","status":"claimed"}`. Invalid, expired, used, or another user's claimed code returns `400 {"reason":"invalid_pair_code"}`.

### `POST /pair/confirm` (Companion)

Request:

```json
{"deviceId":"dev_xxx","deviceSecret":"...","deviceName":"Watson's MacBook Air"}
```

Before claim it returns `409 {"reason":"pair_pending"}`. On success it returns `201 {"deviceToken":"..."}`. The Companion persists this token and authenticates its WSS connection using it. The Relay stores only its hash.

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
    "lastSeenAt":"2026-08-16T06:00:00.000Z"
  }]
}
```

`online` reflects the device WSS connection. `dshStatus` is the Companion's latest local health observation.

### `PATCH /devices/:id` (User)

Request: `{"name":"New name"}`. Returns the renamed device. The caller must own the device.

### `DELETE /devices/:id` (User)

Returns `204`. The Relay immediately revokes the device token and closes the active Companion connection. A later Companion reconnect must receive `auth_fail` and return to pairing.

## WebView authorization and proxy

### `POST /web-ticket` (User)

Request: `{"deviceId":"dev_xxx"}`. Returns `201 {"ticket":"...","expiresIn":60}`. Tickets are single-use, held only in Relay memory or short-lived persistent storage, and are bound to the calling account and device.

The mobile client loads:

```
https://<relay-host>/s/dev_xxx/?ticket=<ticket>
```

On success the Relay consumes the ticket, sets an `HttpOnly; Secure; SameSite=Lax; Path=/s/dev_xxx` session cookie (two-hour lifetime), then serves the proxied DSH response. No bearer token or device secret is passed into the WebView.

All subsequent `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, and WebSocket-upgrade traffic beneath `/s/:deviceId/` uses that cookie. A ticket is never accepted for another device.

## User-facing proxy errors

| HTTP | Body | Mobile meaning |
| --- | --- | --- |
| `401` | `{"reason":"web_session_expired"}` | Request a new web ticket and reload silently |
| `403` | `{"reason":"device_forbidden"}` | The account does not own the device |
| `503` | `{"reason":"device_offline"}` | Computer/Companion is offline |
| `503` | `{"reason":"dsh_offline"}` | Computer is online but local DSH is unavailable |
| `504` | `{"reason":"tunnel_timeout"}` | Retryable connectivity failure |
| `502` | `{"reason":"body_too_large"}` | MVP response-size limitation |

The mobile client must allow navigation inside the Relay origin only. External origins open in the operating system browser.

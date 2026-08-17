# DSH Browser Remote and Relay Admin v1

## Scope

The Relay hosts two separate web surfaces:

- `/app/` is the public iPhone and desktop browser client.
- `/admin/` is an operator-only statistics console, disabled until credentials are configured.

The public browser is a new endpoint for the existing `sealed-tunnel-v1` profile. It is not a plaintext Relay proxy and does not copy or reimplement the DSH product UI. It loads the original loopback DSH Web UI through a browser-side encrypted gateway.

## Browser trust bootstrap

Companion creates a six-digit claim code and a random 32-byte master key, then renders:

```text
https://<relay-origin>/app/#/pair?code=<six-digits>&key=<base64url-key>
```

iPhone Camera recognizes the HTTPS QR and opens Safari. Only `/app/` is sent to the server. The fragment stays inside the browser. After login, the browser claims the code, stores the key under the returned `deviceId` in IndexedDB, removes the fragment from history, and refreshes the device list.

Pairing links are credentials. Users must not paste them into chat, analytics, crash reports, or support tickets. Re-pairing rotates the key. Removing a device deletes the browser key and revokes the Relay device association.

## Browser session gateway

1. Browser requests a one-time `/web-ticket` with its HttpOnly user session.
2. Browser starts the root-scoped DSH Service Worker and transfers the ticket and device key over `postMessage`.
3. Service Worker opens `/client-tunnel` with `dsh-e2ee-v1` and an encoded ticket subprotocol.
4. Service Worker and Companion complete the authenticated E2EE handshake.
5. Navigation to `/remote/:deviceId/` is intercepted locally. HTTP and SSE become encrypted `http_req` and `http_res` envelopes.
6. A bootstrap inserted into the decrypted DSH HTML maps browser WebSocket objects to encrypted `ws_open`, `ws_frame`, and `ws_close` envelopes.

The Service Worker rewrites loopback redirects to the controlled browser origin, keeps upstream cookies in a session-only local jar, and removes upstream CSP before inserting the WebSocket bootstrap. It passes only decrypted DSH content to the controlled DSH page. If Safari reclaims the worker, the secure route ends and the user returns to `/app/` to request a fresh ticket.

The Relay-hosted browser bundle is part of the trusted endpoint. This design protects plaintext from normal Relay routing, persistence, logs, database access, and passive infrastructure observation. It does not protect a user from a malicious or compromised Relay deployment that serves modified JavaScript, because that code executes before the browser can establish E2EE. Users who require protection from the web host itself must use a separately signed native client or extension with pinned code.

## Visible metadata

Relay and Admin can observe:

- account and device association;
- device online and DSH health state;
- access start, last activity, expiry, and end state;
- bounded platform and OS labels;
- ciphertext sizes, timing, and connection counts.

They cannot observe DSH paths, request headers, cookies, bodies, task text, model output, WebSocket payloads, or close reasons inside sealed frames.

## Admin boundary

Operators set `ADMIN_USERNAME` and `ADMIN_PASSWORD` in the deployment secret manager. Empty values disable the admin login. Admin authentication uses a separate signed Cookie and is rate-limited with browser authentication.

The first release exposes only read-only statistics:

- total and newly registered users;
- 30-day active users;
- total, paired, online, and DSH-online devices;
- active and recent access sessions;
- 30-day daily new-user and access-session series;
- at most 50 recent session metadata rows.

No user mutation, password reset, device control, impersonation, tunnel inspection, or content search is available in Admin v1.

## Deployment gates

- HTTPS/WSS is mandatory for public browser use.
- `PUBLIC_RELAY_URL` must be the canonical HTTPS origin.
- `ALLOW_LEGACY_WEB_PROXY` must remain `0`.
- `JWT_SECRET` must meet the existing production strength checks.
- Admin credentials must exist only in deployment secrets.
- The Relay remains a single instance while it uses local SQLite and in-memory connection routing.

# DSH Sealed Tunnel E2EE Profile v1

This profile defines the application-layer encryption used by DSH Remote 0.1.3. The Relay authenticates accounts and devices and routes frames, but never receives the content-encryption key or plaintext DSH traffic.

## Security boundary

The profile protects HTTP methods, paths, headers, bodies, response status, response headers, SSE chunks, WebSocket payloads, and close reasons from disclosure or modification by the Relay. It does not hide account/device associations, online state, connection time, ciphertext length, or traffic timing. Compromised Mobile or Companion endpoints are out of scope.

Version 1 uses a QR-delivered pre-shared key and does not provide forward secrecy. A later profile may add authenticated ephemeral X25519 without changing the inner HTTP/WS envelope.

## Secure pairing

The Companion generates a fresh 32-byte `e2eeMasterKey` locally for every pairing attempt. It is never sent to the Relay. The QR payload is:

```json
{"v":2,"relay":"https://relay.example","code":"482913","e2eeKey":"<unpadded-base64url-32-bytes>"}
```

The Mobile client accepts only QR version 2, validates that `e2eeKey` decodes to exactly 32 bytes, claims `code`, then stores the key in platform secure storage under the authenticated account and returned `deviceId`. The Companion stores the same key only after `/pair/confirm` succeeds, using its existing mode-0600 configuration file.

A six-digit code alone is not a secure E2EE bootstrap and MUST NOT complete production pairing. Unbind, account logout, and re-pair delete the associated key. Existing devices MUST re-pair by scanning a version 2 QR.

## Relay authorization

The Companion advertises `sealed-tunnel-v1` in its `/device` `auth` payload. `POST /web-ticket` succeeds only when the selected device is connected with that capability. Its additive response is:

```json
{
  "ticket":"<one-time-token>",
  "expiresIn":60,
  "accessSessionId":"access_uuid",
  "tunnelUrl":"wss://relay.example/client-tunnel",
  "e2eeRequired":true
}
```

Mobile opens `tunnelUrl` with `Authorization: WebTicket <ticket>`. Tickets MUST NOT be placed in URLs. The Relay consumes a ticket once, binds the socket to its account, device, and access session, and routes only the message types below.

## Canonical encoding

- Binary values use unpadded base64url.
- HMAC and HKDF use SHA-256.
- AEAD is AES-256-GCM with a 16-byte tag appended to ciphertext before base64url encoding.
- JSON used by HMAC/AAD is UTF-8 without whitespace. Authenticated structures are arrays so field order is unambiguous.
- Sequence values are unsigned 64-bit decimal strings on the wire.

## Handshake

Mobile creates 32 random bytes `clientRandom` and sends `client_hello`:

```json
{"v":1,"type":"client_hello","id":"uuid","ts":0,"payload":{"accessSessionId":"access_uuid","clientRandomB64":"...","clientProofB64":"..."}}
```

`clientProof` is:

```text
HMAC(masterKey, utf8(JSON(["dsh-e2ee-client",1,accessSessionId,clientRandomB64])))
```

After verifying the proof, Companion creates 32 random bytes `serverRandom` and returns `server_hello`. `serverProof` is:

```text
HMAC(masterKey, utf8(JSON(["dsh-e2ee-server",1,accessSessionId,clientRandomB64,serverRandomB64])))
```

Both sides compute:

```text
salt = SHA256(utf8(JSON(["dsh-e2ee-salt",1,accessSessionId,clientRandomB64,serverRandomB64])))
c2dKey       = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:c2d:key"), 32)
d2cKey       = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:d2c:key"), 32)
c2dNonceBase = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:c2d:nonce"), 4)
d2cNonceBase = HKDF(masterKey, salt, utf8("dsh-e2ee-v1:d2c:nonce"), 4)
```

Proof failure, timeout, missing keys, malformed base64url, or an unexpected handshake transition closes the secure session. The Relay cannot request a plaintext fallback.

## Sealed frames

The outer envelope remains tunnel version 1:

```json
{"v":1,"type":"sealed","id":"uuid","ts":0,"payload":{"accessSessionId":"access_uuid","seq":"0","ciphertextB64":"..."}}
```

The 12-byte GCM nonce is `nonceBase32 || seq64be`. AAD is:

```text
utf8(JSON(["dsh-e2ee",1,accessSessionId,"c2d|d2c",seq]))
```

The plaintext is one complete existing v1 `http_req`, `http_res`, `http_close`, `ws_open`, `ws_open_ok`, `ws_frame`, or `ws_close` envelope. The Relay MUST NOT receive `channel`, HTTP/WS fields, or payload data outside the ciphertext.

Each direction starts at sequence 0 and increments by exactly one. Duplicate, skipped, reordered, wrong-direction, wrong-session, or unauthenticated frames close the secure session before any plaintext is forwarded.

## Shared known-answer vector

Implementations MUST reproduce this vector before interoperating:

```text
masterKeyB64      = AAECAwQFBgcICQoLDA0ODxAREhMUFRYXGBkaGxwdHh8
accessSessionId   = access_test_vector
clientRandomB64   = ICEiIyQlJicoKSorLC0uLzAxMjM0NTY3ODk6Ozw9Pj8
clientProofB64    = F3mAmAuR30RnLXq7TMUeMNWZquo8GPHVyCxWycTDW80
serverRandomB64   = QEFCQ0RFRkdISUpLTE1OT1BRUlNUVVZXWFlaW1xdXl8
serverProofB64    = IXvvgrKVbAjjW-M2rLOmf-blsUwmbgLa6y79lEj1vNA
```

For the first c2d plaintext below, `seq` is `0` and `ciphertextB64` is:

```json
{"v":1,"type":"http_req","channel":"ch_test","id":"request_1","ts":0,"payload":{"method":"POST","path":"/canary","bodyB64":"c2VjcmV0"}}
```

```text
voqHzLsUCWC__C-NBr-s0t1AshPpTEwNwSoOrhx_ihApnwkPlwKxr7EX28kmqOjoVQus161QnjXyzjxDil_WXgnkvu0pgQfiGV27QIgL97KPe2X0nv9vlzLUmwMll0ipeUo2IZKgM-Rt_WRa_-TyyL9SEkozijz1Z6HBxk1hfLiwQEb602fzHVGQP4Wh3Q2B41mrL_-AGQ
```

## Lifecycle and limits

`client_close` and `device_close` terminate the access session. Relay/device/mobile disconnects close all inner HTTP and WebSocket channels. The Relay enforces ciphertext frame size, byte rate, idle timeout, handshake timeout, and per-device/global secure-session counts. Mobile and Companion enforce inner request, response, channel, and plaintext limits.

The legacy `/s/:deviceId` plaintext proxy is disabled by default. Production returns `426 {"reason":"e2ee_required"}` for clients or devices without this profile; it never silently downgrades.

## Error reasons

| Reason | Meaning |
| --- | --- |
| `e2ee_required` | Client or Companion lacks the mandatory capability/key |
| `invalid_web_ticket` | Ticket is invalid, expired, reused, or bound elsewhere |
| `e2ee_handshake_failed` | Proof, state, encoding, or timeout failure |
| `e2ee_auth_failed` | AEAD authentication, sequence, direction, or session binding failed |
| `secure_tunnel_capacity` | Relay secure-session resource boundary reached |

## Release verification

Tests MUST include one shared known-answer vector, cross-language seal/open, bit-flip rejection, replay/reorder rejection, wrong-session rejection, QR/key lifecycle, loopback capability rejection, and real HTTP/SSE/WebSocket traversal. Relay logs, frames, and SQLite are scanned for canary plaintext.

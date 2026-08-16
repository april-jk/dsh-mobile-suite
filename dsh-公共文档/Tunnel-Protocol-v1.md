# DSH Relay Tunnel Protocol v1

The Companion maintains one outbound WebSocket connection to `wss://<relay-host>/device`. The Relay routes browser requests addressed to `/s/:deviceId/*` over that connection. Business payloads are transported only in memory and are never written to logs or the database.

## Envelope

Every frame is UTF-8 JSON:

```json
{
  "v": 1,
  "type": "http_req",
  "channel": "ch_123",
  "id": "uuid",
  "ts": 1755500000000,
  "payload": {}
}
```

`channel` is required for tunnel messages and absent for control messages. `id` is unique per sender. Unknown additive fields must be ignored.

## Connection lifecycle

1. Companion opens WSS and immediately sends `auth`.
2. Relay validates `deviceId` and `deviceToken`, replies `auth_ok`, and marks device online.
3. Companion sends `ping` every 25 seconds; Relay returns `pong`.
4. A Relay that has no inbound frame for 60 seconds closes the socket.
5. Companion reconnects with backoff `1s, 2s, 5s, 10s, 30s`, re-authenticates, and sends its current DSH state.

Authentication frame:

```json
{"v":1,"type":"auth","id":"uuid","ts":1755500000000,"payload":{"deviceId":"dev_xxx","deviceToken":"..."}}
```

Failure response: `{"v":1,"type":"auth_fail","id":"uuid","ts":0,"payload":{"reason":"invalid_device_token"}}`. Success response uses `auth_ok`.

## Status and events

Companion sends `status` whenever local DSH health changes:

```json
{"v":1,"type":"status","id":"uuid","ts":1755500000000,"payload":{"dsh":"online"}}
```

The Relay persists only status metadata. `event` is reserved for future DSH event collection and persists at most 50 event metadata records per device. It does not affect v1 remote UI availability.

## HTTP forwarding

Relay to Companion:

```json
{
  "v":1,
  "type":"http_req",
  "channel":"ch_a1b2",
  "id":"uuid",
  "ts":1755500000000,
  "payload":{"method":"POST","path":"/api/tasks","headers":{"content-type":"application/json"},"bodyB64":"eyJwcm9tcHQiOiJoZWxsbyJ9"}
}
```

The path is the suffix after `/s/:deviceId` for the bootstrap request, or the original absolute DSH path after cookie authorization. It always begins with `/` and must not be interpreted as a URL. The Companion forwards it only to `http://127.0.0.1:<dshPort>`, rewrites `Host`, `Origin`, and `Referer` for that local authority, and removes forwarded proxy identity headers.

Responses are streamed as an initial header frame, zero or more body frames, and one final frame:

```json
{"v":1,"type":"http_res","channel":"ch_a1b2","id":"uuid","ts":1755500000000,"payload":{"status":200,"headers":{"content-type":"text/event-stream"},"bodyB64":"","seq":0,"final":false}}
```

Each following frame increments `seq`. `bodyB64` may be empty. `final:false` keeps the Relay response open; exactly one `final:true` frame ends it. This carries DSH boot responses and SSE without buffering the entire response.

If the browser closes an in-flight request, Relay sends `http_close` on the same channel. Companion destroys the matching loopback request and releases it. A late `http_res` for a closed channel is ignored.

## WebSocket forwarding

1. Relay sends `ws_open {channel, path, headers}`.
2. Companion opens a local DSH WebSocket and responds `ws_open_ok`; failure uses `ws_close`.
3. Either side sends binary-safe frames with `ws_frame {channel, dataB64, opcode}` where `opcode` is `1` for text or `2` for binary.
4. Either side closes using `ws_close {channel, code, reason}`.

The Relay maps a browser WebSocket upgrade to one channel. It must enforce the same WebView cookie authorization before creating the channel. The Companion must close all local channels when the Relay socket closes.

## Resource boundaries

The v1 envelope is unchanged by resource enforcement. The public MVP Relay defaults to a 4 MiB incoming WebSocket frame limit, 2 MiB forwarded HTTP request limit, and 32 MiB total HTTP response limit. It accepts at most 32 HTTP and 16 WebSocket tunnel channels per device, subject to lower global capacity limits. An oversized WebSocket frame closes only that connection with code `1009`; it must not terminate the Relay process.

When an HTTP channel exceeds the response limit, Relay sends `http_close` to Companion and ends the browser response. Companion must release the loopback request after `http_close`. Responses for a channel owned by another authenticated device are ignored.

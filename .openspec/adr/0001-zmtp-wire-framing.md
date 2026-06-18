# ADR-0001: ZMTP 3.x Wire Protocol Design

**Status:** Accepted
**Date:** 2026-06-18
**Context:** pkg-zmq implements ZeroMQ messaging in pure MVL without linking libzmq. The wire protocol must be compatible with pyzmq, zmq.rs, and cppzmq so that MVL services can participate in existing ZeroMQ deployments. This ADR documents the framing, greeting, and handshake design choices.

## Decision

**The ZMTP 3.x framing layer (`src/zmtp.mvl`) implements the minimum viable subset of RFC 23 (ZMTP 3.0) and RFC 37 (ZMTP 3.1) required for NULL-auth interop.**

Two framing modes coexist:

### Mode A: Simplified framing (`src/zmq.mvl`)

Used by the original REQ/REP, PUB/SUB, and PUSH/PULL implementations that follow the connection-per-message model:

```
[4 bytes big-endian u32 length][N bytes body]
```

This is NOT ZMTP-compatible. It is simpler to implement and test, and remains the default for MVL-to-MVL communication where libzmq interop is not needed.

### Mode B: ZMTP 3.x framing (`src/zmtp.mvl`)

Used by the `*_zmtp` variants that wire-interoperate with pyzmq/libzmq. Three structural elements:

**Greeting (64 bytes):**
```
Bytes  0–9:  Signature  0xFF [8 zero bytes] 0x7F
Bytes 10–11: Version    major=0x03, minor=0x01  (ZMTP 3.1)
Bytes 12–31: Mechanism  "NULL" [16 zero bytes]
Byte  32:    as-server  0x00 (client) or 0x01 (server)
Bytes 33–63: Filler     31 zero bytes
```

The greeting is exchanged synchronously before any command or message frames. `make_greeting` builds it; `validate_greeting` checks bytes 0, 9 (signature), byte 10 (version ≥ 3), and bytes 12–15 (mechanism = "NULL").

**Frame format:**
```
[flags: 1 byte]
  bit 0 (0x01): MORE    — more frames follow in this message
  bit 1 (0x02): LONG    — 8-byte size field (vs 1-byte for body < 256 bytes)
  bit 2 (0x04): COMMAND — command frame (vs message frame)
[size: 1 or 8 bytes, big-endian]
[body: `size` bytes]
```

`read_frame_raw` and `write_frame_raw` implement the codec. Both are `total fn` — all branches terminate with a `Result` value, no unbounded reads.

**READY command (NULL auth handshake):**

After the greeting, both peers exchange a command frame (flags=0x04) with body:

```
[1-byte name-length=5]["READY"]
[1-byte key-length=11]["Socket-Type"]
[4-byte BE value-length][socket-type-name]
```

Socket-type names are: "REQ", "REP", "PUB", "SUB", "PUSH", "PULL". Additional properties (e.g., "Identity") are scanned and skipped by `parse_socket_type_property` until "Socket-Type" is found.

**PING/PONG heartbeat (ZMTP 3.1):**

When `zmtp_recv_message` encounters a PING command frame, it calls `handle_command` which responds with PONG (echoing the context bytes after the 2-byte TTL, omitting the TTL itself) and loops to receive the next frame. This keeps the connection alive without surfacing heartbeat frames to the application.

## Rationale

- **64-byte greeting** is a fixed-size structure. A fixed read avoids the complexity of streaming parse.
- **1-byte vs 8-byte size field** is indicated by the LONG flag. The 8-byte field covers frames up to 2^63 bytes; MVL enforces a 64 MB cap in both `read_frame_raw` and `decode_frame` to prevent DoS via crafted length fields.
- **REQ/REP envelope** (empty delimiter + body) is handled transparently: `zmtp_send_message` prepends the empty frame; `zmtp_recv_message` skips delimiter and command frames, returning only the final non-MORE body. Application code sees no envelopes.
- **SUBSCRIBE command** (RFC 37): `zmtp_send_subscribe` sends a ZMTP 3.1 command frame; `zmtp_recv_subscribe` also accepts the ZMTP 3.0 fallback (message frame with 0x01 prefix) for compatibility with older peers.

## 64 MB frame size cap

Both framing layers enforce a 64 MB limit:

```mvl
if size < 0 || size > 67108864 {
    Err(ZmqError::FrameError("frame too large"))
}
```

67 108 864 = 64 × 1024 × 1024. This matches the libzmq default and is large enough for practical payloads while bounding memory allocation per frame.

## Consequences

- Any ZMTP 3.x peer using NULL security and one of the six supported socket types interoperates with pkg-zmq out of the box.
- PLAIN and CURVE security mechanisms are not supported; connections using them are rejected at `validate_greeting` (mechanism != "NULL").
- ZMTP 2.x (no greeting exchange) is not supported.
- The simplified framing mode (`encode_frame`/`decode_frame`) remains available for MVL-to-MVL use cases that do not need libzmq interop.

## Connected to

- spec 001-zmtp-wire: REQ 1 (greeting), REQ 2 (NULL auth), REQ 3 (frame codec), REQ 4 (multi-frame)
- ADR-0002: IFC/taint design — `read_bytes` detaints with "ZMTP-PARSE"
- ADR-0003: Totality policy — `read_frame_raw`, `write_frame_raw`, `make_greeting`, `validate_greeting` are all `total fn`
- RFC 23: https://rfc.zeromq.org/spec/23/ (ZMTP 3.0)
- RFC 37: https://rfc.zeromq.org/spec/37/ (ZMTP 3.1)

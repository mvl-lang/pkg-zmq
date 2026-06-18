# ADR-0002: IFC/Taint Label Design

**Status:** Accepted
**Date:** 2026-06-18
**Context:** MVL's information-flow control (IFC) system tracks untrusted data through the program using `Tainted[T]` wrappers. pkg-zmq receives data from arbitrary remote peers and must label it consistently so that auditors can trace where tainted data is used, where it is trusted, and under what justification.

## Decision

**All bytes received from the network are `Tainted[String]`. Each `relabel trust` operation carries a distinct audit tag that documents the security reasoning at that boundary.**

### Audit tags used in pkg-zmq

| Tag | Location | Meaning |
|-----|----------|---------|
| `ZMTP-PARSE` | `zmtp.mvl::read_bytes` | Raw bytes detainted for protocol structure parsing only (frame flags, size field, greeting bytes). The bytes are structural, not user content. |
| `ZMQ-FRAME-PARSE` | `zmq.mvl::decode_frame` | Raw `Tainted[String]` from `tcp_read` detainted for the 4-byte length header extraction. Length bytes are compared as integers; no user content is interpreted. |
| `ZMQ-RECV` | `zmq.mvl::decode_frame`, `zmtp.mvl::zmtp_recv_message` | The message body after structural parsing. This is the boundary where untrusted user content is re-tainted and handed to the application. |
| `ZMQ-SUB-FILTER` | `pubsub.mvl::sub_recv`, `sub_accept_loop` | Topic component extracted from the PUB/SUB payload for the prefix filter check. The filter check is the security boundary: only after the topic is verified to match the filter is the payload re-tainted and forwarded. |
| `ZMQ-SUB-TOPIC` | `pubsub.mvl::sub_topic` | Topic string extracted from a `Tainted[String]` payload. Returned as plain `String` — topic is structural (routing metadata), not free-form user content. |
| `ZMQ-SUB-BODY` | `pubsub.mvl::sub_body` | Message body after splitting on `\n`. Re-tainted as `Tainted[String]` because the body is user-supplied content. |

### Trust flow diagram

```
tcp_read / tcp_read_exact
        │
        │  Tainted[String]  (raw network bytes)
        ▼
  decode_frame / read_bytes
        │  relabel trust(..., "ZMQ-FRAME-PARSE" | "ZMTP-PARSE")
        │  — structural bytes only (length, flags, greeting fields)
        │
        ▼  plain String  (structure)
  boundary check (length, size, signature bytes, mechanism)
        │
        │  relabel taint(..., "ZMQ-RECV")
        ▼
  Tainted[String]  ← application receives this
```

For PUB/SUB, `sub_recv` adds an additional hop:

```
Tainted[String]  (ZMQ-RECV from decode_frame)
        │
        │  relabel trust(..., "ZMQ-SUB-FILTER")
        │  — for topic prefix check only
        ▼
  topic_passes_filter(plain, filter)
        │  (boolean result)
        │  if passes:
        │  relabel taint(..., "ZMQ-RECV")
        ▼
  Tainted[String]  — forwarded to handler
```

## Rationale

### Why separate ZMQ-FRAME-PARSE from ZMQ-RECV?

The length bytes and the body bytes are trusted for different reasons:

- Length bytes (4 bytes, Mode A) or size/flags bytes (1–9 bytes, Mode B) are trusted because they are **structural** — the packet cannot be interpreted without them, and they are compared as integers against known limits. There is no injection risk in reading an integer.
- The body bytes are trusted (as `ZMQ-RECV`) only after the boundary check confirms the declared length matches the actual byte count. At that point the body is validated for structural integrity but not for content. It is re-tainted immediately.

### Why ZMTP-PARSE instead of ZMQ-FRAME-PARSE for zmtp.mvl?

`zmtp.mvl` reads many structural byte sequences: greeting bytes (checked against constants), command name bytes (compared to "READY", "PING", "PONG"), property key bytes (compared to "Socket-Type"), property value bytes (compared to socket-type names). All of these are structural parsing, not user content. `ZMTP-PARSE` groups them under a single audit tag that signals "this trust operation is protocol structure, not application data."

### Why is `sub_topic` returned as plain String?

The topic is routing metadata chosen by the publisher. After the `ZMQ-SUB-FILTER` boundary check (which verifies the topic matches the subscriber's filter), the topic string has been structurally validated. Returning it as `String` rather than `Tainted[String]` signals that it is safe to use as a routing key or log label — it is not arbitrary user payload. The body (`sub_body`) remains `Tainted[String]` because it is free-form user content.

## Consequences

- Auditors can search for each audit tag to find every site where untrusted data is trusted.
- Adding a new trust boundary requires a new, uniquely named audit tag with a documented justification.
- Application code that calls `rep_serve`, `puller_serve`, or `sub_serve` always receives `Tainted[String]` and must apply its own `relabel trust` with a context-specific tag before any security-sensitive use.
- The six audit tags are the full IFC interface of pkg-zmq — no other trust operations exist in the package.

## Connected to

- spec 001-zmtp-wire: REQ 9 (IFC labels preserved)
- ADR-0001: ZMTP wire framing — `read_bytes` is where ZMTP-PARSE is applied
- MVL IFC documentation: `std.ifc.{Tainted}`, `relabel trust`, `relabel taint`

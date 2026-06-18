# Changelog

All notable changes to pkg-zmq will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.3.0] - 2026-06-18

### Added
- Makefile: `coverage` target — `mvl test src/ --coverage` for behavioral branch coverage report
- Makefile: `version` target — prints current version from `mvl.toml`
- `.openspec/adr/0001-zmtp-wire-framing.md` — ADR documenting ZMTP 3.x wire protocol design (4-byte length prefix, greeting structure, frame flags, READY handshake)
- `.openspec/adr/0002-ifc-taint-design.md` — ADR documenting IFC/taint label strategy (ZMQ-RECV, ZMQ-FRAME-PARSE, ZMQ-SUB-FILTER, ZMTP-PARSE)
- `.openspec/adr/0003-totality-policy.md` — ADR documenting totality policy (actor loops legitimately partial; ZMTP helpers total)

## [0.2.0] - 2026-06-09

### Changed
- Adapt to MVL `String.byte_at` API change: now returns `Option[Byte]` instead of `Byte` (#1263)
- All 53 `byte_at(i).to_int()` call sites updated to `byte_at(i).unwrap_or(from_int(0)).to_int()`
- Bumped `requires-mvl` to `>=0.190.0`

## [0.1.0] - 2026-05-31

### Added
- Initial release -- ZeroMQ-style messaging for MVL
- REQ/REP pattern (`pkg.zmq.reqrep`): `req_send`, `rep_serve`
- PUB/SUB pattern (`pkg.zmq.pubsub`): `pub_send`, `sub_serve` with topic filtering
- PUSH/PULL pattern (`pkg.zmq.pushpull`): `push_send`, `puller_serve` for work distribution
- Simplified ZMTP wire protocol: 4-byte big-endian length prefix + body
- `encode_frame` / `decode_frame` -- pure functions, no I/O
- 64 MB frame size limit (DoS protection)
- `ZmqAddr` -- refined type: host non-empty, port 0..65535
- `parse_zmq_addr` -- parse `"tcp://host:port"`, `"tcp://*:port"`, `"host:port"`
- `zmq_bind` -- parse + TCP listen in one step
- `send_one` -- connect-write-close for fire-and-forget semantics
- `is_transient_accept_error` -- retry logic for accept loops
- IFC: all received bytes `Tainted[String]`; detainted at `ZMQ-FRAME-PARSE`, body re-tainted `ZMQ-RECV`
- ZMTP 3.x wire compatibility: PING/PONG heartbeat, interop with pyzmq/libzmq
- Pure MVL -- no `bridge.rs`, no `extern` blocks

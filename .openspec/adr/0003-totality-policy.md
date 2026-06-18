# ADR-0003: Totality Policy — Explicit `total fn`, Legitimate `partial fn`

**Status:** Accepted
**Date:** 2026-06-18
**Context:** MVL infers totality (`total*`) for functions that have no unbounded loops and no calls to `partial fn`. pkg-zmq has two categories of functions: pure protocol helpers that provably terminate, and actor loops / receive loops that run until the peer closes or a fatal error occurs. The question is how to annotate and audit both categories.

## Decision

Same policy as pkg-http ADR-0002 and pkg-sqlite ADR-0003: **all terminating functions carry explicit `total fn`. No implicit totality (`total*`) is permitted in source files.**

Actor loops and receive loops that block indefinitely are legitimately `partial fn` and remain so by design.

## Application to pkg-zmq

### `total fn` functions (provably terminating)

All pure protocol helpers in `src/zmtp.mvl` and `src/zmq.mvl` are `total fn`:

| Function | Why total |
|---|---|
| `socket_type_name` | Exhaustive enum match, no loops |
| `parse_socket_type` | Exhaustive if-else chain, no loops |
| `bval`, `zeros_8`, `zeros_15`, `zeros_16`, `zeros_31` | Constructors, no loops |
| `make_greeting` | String concatenation, no loops |
| `validate_greeting` | Length checks and byte comparisons, no loops |
| `make_ready_body` | String construction, no loops |
| `is_ping`, `ping_context` | Bounds checks and substring, no loops |
| `handle_command` | Calls only `total fn` callees (`is_ping`, `ping_context`, `write_frame_raw`) |
| `read_bytes`, `write_bytes` | Single I/O call + Result mapping, no loops |
| `read_frame_raw`, `write_frame_raw` | Two I/O calls with conditional, no loops |
| `encode_frame`, `decode_frame` | Pure arithmetic and substring, no loops |
| `parse_zmq_addr` | String parsing with bounded branching, no loops |
| `map_net_err`, `zmq_error_msg` | Enum match, no loops |
| `is_transient_accept_error` | Enum match, no loops |
| `zmq_bind`, `send_one` | Call `total fn` callees + single I/O, no loops |
| `zmtp_send_message`, `zmtp_send_body` | Two `write_frame_raw` calls, no loops |
| `zmtp_send_subscribe` | String construction + `write_frame_raw`, no loops |
| `topic_passes_filter` | String search + `starts_with`, no loops |
| `sub_topic`, `sub_body` | String split, no loops |
| `puller_recv` | Single `tcp_accept` + `tcp_read`, no loops |

### `partial fn` functions (legitimately unbounded)

Several categories of functions are legitimately `partial fn` because they loop until an external event (peer disconnect, fatal accept error):

**Actor loops** — `while true` loops that wait for incoming work:

| Function | Why partial |
|---|---|
| `rep_accept_loop` | Infinite accept loop — returns only on fatal accept error |
| `rep_zmtp_accept_loop` | Same |
| `sub_accept_loop` | Same |
| `pub_zmtp_accept_loop` | Same |
| `puller_accept_loop` | Same |
| `pull_zmtp_accept_loop` | Same |

**Receive loops** — `while true` loops that wait for frames on a persistent connection:

| Function | Why partial |
|---|---|
| `zmtp_recv_message` | Loops to skip delimiter/command frames; terminates when non-MORE data frame arrives or on error — but the number of skipped frames is unbounded |
| `zmtp_recv_subscribe` | Loops to skip frames until SUBSCRIBE command; same argument |
| `zmtp_exchange_loop` | Loops request/reply until peer disconnects |
| `sub_zmtp_recv_loop` | Loops until peer disconnects |
| `pull_zmtp_recv_loop` | Same |

**ZMTP handshake functions** — call `partial fn` callees:

| Function | Why partial |
|---|---|
| `zmtp_handshake_server` | Calls `read_bytes` (total) and `read_frame_raw` (total), BUT also calls `parse_ready_body` which calls `parse_socket_type_property` — which is `partial fn` due to recursive self-calls on unknown properties |
| `zmtp_handshake_client` | Same |

**Note on `parse_socket_type_property`:** This function is `partial fn` because it recurses on unknown properties (`parse_socket_type_property(body, ve)`). MVL's termination checker cannot verify that this recursion terminates without a `decreases` clause. In practice it terminates because `ve > offset` always (each property consumes at least 5 bytes) and the body is finite. A future improvement would add `decreases body.len() - offset` to make this `total fn`.

**Top-level serve functions** — call `partial fn` callees:

`rep_serve`, `rep_serve_zmtp`, `sub_serve`, `pub_serve_zmtp`, `puller_serve`, `pull_serve_zmtp`, `sub_connect_zmtp`, `push_connect_zmtp`, `req_request_zmtp` are all `partial fn` because they call accept or receive loops.

**Actor methods** — `Publisher::publish`, `Pusher::push`, `Publisher::subscribe`, `Pusher::add_worker` carry no totality keyword (actor scheduling is partial by nature, as in pkg-http ADR-0002).

## Totality of ZMTP helpers

The key insight is that `read_frame_raw` and `write_frame_raw` are `total fn` even though they perform I/O. Totality and effects are orthogonal in MVL: `total fn ! Net` means "terminates with certainty, may perform network I/O." The `! Net` effect is declared separately from the termination guarantee.

This makes the protocol helpers composable: any function that calls only `total fn ! Net` callees and has no loops is itself `total fn ! Net`.

## Target state

`make assurance` should report `total fn: N (N explicit, 0 implicit)`. Any new function added without an explicit totality keyword will appear as `total*` and violate the policy.

## Consequences

- `make assurance` will flag any new function missing an explicit totality keyword as `total*`.
- Reviewers should reject PRs that introduce implicit totality.
- The `partial fn` category for receive loops is intentional and documented here; it is not a quality failure.
- `parse_socket_type_property` is a known candidate for upgrade to `total fn` via a `decreases` clause (tracked as future work).

## Connected to

- MVL Req 3 (Totality) and Req 8 (Termination): verified by `mvl assurance`
- ADR-0001: ZMTP wire framing — all ZMTP helpers that are `total fn`
- ADR-0002: IFC/taint design — `read_bytes` and `write_bytes` are `total fn ! Net`
- pkg-http ADR-0002: same explicit `total fn` policy, established first
- pkg-sqlite ADR-0003: same policy with `decreases` clause example

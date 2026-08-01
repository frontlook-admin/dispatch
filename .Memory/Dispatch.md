# Dispatch repository memory

## Purpose

`dispatch-proxy` is a Rust 2021 cross-platform SOCKS proxy. It balances outbound TCP connections across configured local network interfaces or source IP addresses using weighted round-robin selection.

It balances connections, not packets. A single TCP connection stays on one selected interface; clients that open multiple connections can combine bandwidth.

## Commands

- `dispatch list` lists usable network interfaces and addresses.
- `dispatch start <address>[/weight]` starts the proxy.
- Default listener: `127.0.0.1:1080`.
- `--ip` changes the listener address; `--port` changes the listener port.
- `--debug` writes tracing logs to stdout; otherwise logs go to the application data directory.

## Runtime flow

1. Resolve each configured argument as an interface name or explicit IP address.
2. For interface names, retain the first usable IPv4 and first usable IPv6 address.
3. Accept client TCP connections concurrently with Tokio.
4. Parse SOCKS4 or SOCKS5 requests.
5. Resolve SOCKS domain targets locally and use the first resolved address.
6. Select a source address using independent IPv4/IPv6 weighted round-robin state.
7. Bind an outbound socket to the selected source address, connect to the target, and report SOCKS success/failure.
8. Pipe traffic bidirectionally until either side closes.

## Supported and unsupported behavior

Supported:

- SOCKS5 `NOAUTH`.
- SOCKS5 `CONNECT`.
- SOCKS4 `CONNECT`.
- IPv4 and IPv6, with target/source family matching.
- Interface names and explicit IP addresses.
- Weighted syntax such as `10.0.0.1/7`.

Unsupported:

- SOCKS authentication.
- HTTP proxy protocol.
- SOCKS `BIND` and `UDP ASSOCIATE`.
- Packet-level balancing, connection failover, or retries across interfaces.

## Important constraints

- An IPv4 target requires at least one configured usable IPv4 source; IPv6 likewise requires IPv6.
- `dispatch list` filters loopback, link-local, and addresses that cannot bind locally.
- Exposing the listener beyond localhost creates an unauthenticated proxy.
- Windows-specific connect error mapping is incomplete; source comments mark this as TODO.

## Main implementation files

- `README.md` — usage, examples, rationale, and behavior.
- `src/main.rs` — CLI parsing and command dispatch.
- `src/list.rs` — interface listing.
- `src/net.rs` — local address filtering and socket binding.
- `src/dispatcher/weighted_rr.rs` — address resolution and weighted round-robin dispatcher.
- `src/socks.rs` — SOCKS4/SOCKS5 handshake, DNS lookup, connect, and status responses.
- `src/server.rs` — listener, per-connection tasks, and traffic piping.
- `src/debug.rs` — tracing, file logging, and panic reporting.

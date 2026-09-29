# slopio

An async runtime for Rust built directly on io_uring, and the network stack on
top of it: TLS, HTTP/2, gRPC, WebSocket and DNS. Linux only, thread-per-core,
no tokio.

I started it to see what a runtime looks like when io_uring is the design
rather than a backend hidden behind epoll-shaped traits. It has since become
the place where I try things out, from the kernel ring up to the protocols.

## The ideas

- **One thread, one ring, nothing shared.** Each worker owns its tasks,
  sockets and buffers. Futures need not be `Send`, and there is no lock in the
  hot path. Workers message each other through the ring itself.
- **io_uring's features are the API.** The kernel picks the receive buffer,
  one submission delivers a stream of accepts or reads, and memory is
  registered once and sent without a copy. None of this goes through
  `AsyncRead` and `AsyncWrite`.
- **No hidden copy, no hidden allocation.** Every I/O call says which buffer
  it uses and who owns it afterwards.
- **Correct by construction.** A cancelled operation never leaves the kernel
  writing into freed memory, a descriptor closes only after its last
  completion, and a broken invariant aborts the process instead of corrupting
  it. The runtime is tested against a simulated kernel, and those suites also
  run under Miri.
- **TLS in the kernel.** The handshake runs in `rustls`, then the record
  layer moves to the kernel (kTLS), so encrypted reads and writes stay plain
  io_uring operations.

## Repositories

| Repository | What it is |
|---|---|
| [slopio](https://github.com/slopio/slopio) | The runtime: tasks, I/O driver, timers, channels, messaging between workers, telemetry, deterministic simulator. |
| [slopio-tls](https://github.com/slopio/slopio-tls) | TLS 1.3 handshake with `rustls`, then the record layer handed to the kernel. |
| [slopio-h2](https://github.com/slopio/slopio-h2) | HTTP/2 server and client, with gRPC on top. HTTP/2 only; h2spec passes clean, strict suite included; defences against the published HTTP/2 denial-of-service attacks. |
| [slopio-ws](https://github.com/slopio/slopio-ws) | WebSocket (RFC 6455), plain or over kTLS. One copy per byte, batched writes, no allocation once a connection is warm. |
| [slopio-dns](https://github.com/slopio/slopio-dns) | DNS client over UDP. Many queries in flight on one socket, and protection against forged responses. |

`slopio-h2` and `slopio-ws` get TLS from `slopio-tls`, and `slopio-h2`
resolves names with `slopio-dns`.

## Status

Experimental. APIs change without notice, nothing is on crates.io yet, and
some subsystems are still stubs. Depend on the crates by git.

## Requirements

- **Linux 6.1 or later.** 6.5 is needed to wait on child processes. There is
  no epoll fallback and no macOS or Windows support. WSL2 works if its kernel
  is recent enough.
- **io_uring allowed.** The default seccomp profiles of Docker and containerd
  block it, so containers need a custom profile.
- **kTLS for TLS.** A server or client asked for TLS on a kernel without it
  refuses to start rather than fall back to userspace.
- **Rust 1.85 or later** (edition 2024), on stable.

The details are in the runtime's
[requirements](https://github.com/slopio/slopio/blob/main/docs/requirements.md).

## License

Dual-licensed under MIT or Apache-2.0, at your option.

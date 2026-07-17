# Dovenix OS

A modern, secure, POSIX-compatible operating system built in Rust — microkernel-grade
isolation enforced by modern hardware security features, with monolithic-kernel-grade
performance as the target.

> **Status: pre-alpha, design phase.** Nothing boots yet. The current deliverables are
> design documents; see [The Dovenix Book](docs/).

## Vision

Dovenix aims to be a daily-driver OS for desktop, server, and eventually mobile, with
security as the first design input rather than a retrofit. The design center:

- **Microkernel-equivalent security without the full IPC tax.** Drivers and system
  services run as isolated userspace components, but the design leans on modern hardware
  (IOMMU/DMA isolation, virtualization extensions, MPK/PKS, ARM PAC/BTI/MTE, CET,
  measured/secure boot) to enforce boundaries cheaply instead of paying for message
  passing everywhere.
- **Zircon-inspired, not Zircon-derived.** The kernel follows the pragmatic
  "microkernel-ish" philosophy of capability handles and a small-but-not-ascetic kernel
  boundary, implemented fresh in Rust under permissive licenses.
- **Linux as the performance benchmark.** Speed and resource efficiency are measured
  against the Linux kernel, not against other microkernels.

## Goals (in priority order)

1. **Security** — hardware security features used from the ground up; verified/secure boot.
2. **Stability** — component isolation, crash-restart of drivers and services.
3. **Speed and resource efficiency** — benchmarked against Linux.
4. **Live upgradability** — everything above the kernel upgrades live with rollback;
   kernel updates via near-instant reboot with serialized state handoff (kexec-style).
5. **Testability** — end-to-end and integration tests only; every component is testable
   by booting a partial system in a VM.
6. **Modularity** — components are developed, tested, and shipped independently, and the
   OS is adaptable to specific use cases (server, desktop, mobile).

## Hardware strategy

Real-hardware daily-driving as fast as possible, via a three-tier driver strategy:

1. **Linux driver VM** (day one): an unmodified Linux runs in an isolated VM domain and
   lends its drivers (GPU, Wi-Fi, odd USB) to Dovenix. Legally mere aggregation — no
   license entanglement.
2. **Ported GPL drivers**: individual Linux drivers ported into isolated,
   clearly-marked GPL-2.0-only driver processes ("GPL islands") that speak the
   [driver wire protocol](docs/src/design/driver-wire-protocol.md).
3. **Native drivers** written from hardware specs (VirtIO, NVMe, xHCI, AHCI, …) under
   the project's permissive license — the end state.

## Repository layout

This is a monorepo, structured so components can be split out later without surgery.
Each top-level directory is an independent unit with its own README and license scope:

| Directory | Contents |
|---|---|
| [`docs/`](docs/) | The Dovenix Book (mdBook) — design docs, this is where the project currently lives |
| [`kernel/`](kernel/) | The Dovenix kernel (Rust, `no_std`) |
| [`libs/`](libs/) | Shared libraries: IPC, driver SDK, wire-protocol codecs |
| [`servers/`](servers/) | Userspace system servers: device manager, VFS, network, POSIX layer |
| [`drivers/`](drivers/) | Device drivers — `native/` (permissive) and `ported/` (GPL islands) |
| [`tools/`](tools/) | Host-side tooling: image builder, test runners |
| [`tests/`](tests/) | End-to-end and integration test suites |

## Documentation

The design docs are an [mdBook](https://rust-lang.github.io/mdBook/):

```sh
cd docs && mdbook serve --open
```

Start with the [Driver Wire Protocol](docs/src/design/driver-wire-protocol.md) — it is
the load-bearing interface of the whole system (license boundary, fault isolation,
live update, and testability all hang off it).

## License

Dovenix is dual-licensed under **Apache-2.0 OR MIT**, at your option — see
[LICENSE-APACHE](LICENSE-APACHE) and [LICENSE-MIT](LICENSE-MIT).

**The one deliberate exception:** drivers ported from Linux live under
`drivers/ported/` and are **GPL-2.0-only**, isolated at a process boundary. No code
outside `drivers/ported/` may be derived from GPL sources. The full policy, including
why every linkable library must remain MIT-compatible, is in
[docs/src/design/licensing.md](docs/src/design/licensing.md) and
[CONTRIBUTING.md](CONTRIBUTING.md).

Unless you explicitly state otherwise, any contribution intentionally submitted for
inclusion in Dovenix by you shall be dual-licensed as above, without any additional
terms or conditions.

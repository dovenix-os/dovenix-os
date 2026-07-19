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
- **POSIX is first-class, unlike Fuchsia.** Existing Linux/BSD/macOS software must
  port without unnecessary change (`./configure && make`, zero or trivial patches).
  Where the Zircon model conflicts with correct POSIX (`fork`, signals, filesystem
  semantics), POSIX wins at the boundary; the capability model stays underneath.
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

Alongside these, seven **first-class platform capabilities** are designed in from the
start (see [Goals](docs/src/vision/goals.md)) — each cheap if native, miserable if
retrofitted:

- **Hardware virtualization** — the kernel is a type-1 hypervisor by construction,
  exposed as a public API, KVM-class in role.
- **Containerization** — namespace maps + resource-budgeted jobs, OCI-compatible:
  the docker/podman experience without retrofitted kernel namespaces.
- **Agent operation** — AI agents run with a subset of the user's authority and
  sub-agents with a subset of that, enforced at every depth by the capability model
  itself; every agent run is recordable and exactly replayable for audit.
- **Easy porting (POSIX-first)** — source compatibility with zero or trivial
  patches, per the vision above.
- **Deterministic development and debugging** — scripted thread interleavings in
  tests, component record/replay, whole-system deterministic simulation, a
  production flight recorder.
- **Real-time capability** — priority inheritance across IPC, budgets, core
  shielding; the acceptance benchmark is sub-millisecond pro-audio with zero XRuns.
- **Proper hardware integration** — ACPI, sleep/resume, battery, thermal — riding
  the same driver-lifecycle machinery as live update.

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
| [`plans/`](plans/) | Implementation plans — working documents that sequence the work; the book stays normative |
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

Start with the [load-bearing principles index](docs/src/design/principles.md) —
every principle that resolves design questions across the system, one line each,
linking to where it is specified. The single most load-bearing *interface* is the
[Driver Wire Protocol](docs/src/design/driver-wire-protocol.md): license boundary,
fault isolation, live update, and testability all hang off it.

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

# Roadmap

> **Status: draft.** Milestones, not dates. Each milestone is defined by what its e2e
> test suite can demonstrate in CI.

## M0 — Design (current)

- Driver wire protocol v0 specified.
- Architecture overview stable enough to code against.
- Licensing policy locked before first code lands.

## M1 — Boots in QEMU

- UEFI boot via Limine → kernel → root server on x86_64.
- Kernel: address spaces, threads, capability handles, channels, shared-memory
  objects, interrupts.
- Serial console. First e2e test: boot, run harness, power off, assert transcript.

## M2 — VirtIO userland

- Device manager (supervisor) + driver hosts speaking wire protocol v0.
- Native VirtIO drivers: block, net, console, input.
- First crash-restart demonstration: kill virtio-blk mid-I/O, driver restarts,
  in-flight operations complete or fail cleanly per protocol semantics.

## M3 — POSIX layer

- mlibc (or fork) port; VFS server; initial FS (read-only first, then writable).
- Network stack (candidate: port of a permissively-licensed stack, e.g. smoltcp).
- Shell and coreutils (BSD-derived) run. The system is self-hosting for tests.

## M4 — Real hardware, Linux driver VM

- Boots on a real desktop (NVMe + xHCI native drivers written from spec).
- EL2/VT-x driver VM runs unmodified Linux, lending GPU/Wi-Fi/USB devices to Dovenix
  over virtio transports.
- This is the "daily-driver on my desk" milestone.

## M5 — Live update

- Live upgrade of a running driver and a running server with rollback, exercising the
  quiesce/serialize/restore path of the wire protocol under load.
- Kernel swap via serialized state handoff; measured downtime target: sub-second.

## Later

- ARM64 port. Wayland compositor. First ported GPL drivers where the VM hop hurts.
  Mobile exploration.

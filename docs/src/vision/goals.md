# Goals and Non-Goals

## Goals, in strict priority order

Trade-offs between goals are resolved by this ordering, not case-by-case debate.

### 1. Security

Modern processor hardware security features are design inputs, not retrofits:

- **Boot**: UEFI Secure Boot, measured boot (TPM), a verified chain from firmware to
  every system component.
- **Memory isolation**: separate address spaces for all drivers/servers; IOMMU/SMMU
  mandatory for anything that touches DMA; MPK/PKS for cheap intra-address-space
  compartments where a full process is too expensive.
- **CPU-level mitigations as baseline**: SMEP/SMAP, CET shadow stacks and IBT on x86;
  PAC, BTI, and MTE on ARM64.
- **Virtualization as an isolation tool**: VT-x/EPT and EL2 host untrusted driver
  domains (including the Linux driver VM).
- **Capability-based authority**: no ambient authority; every resource is reached
  through an explicitly granted, unforgeable handle.
- **Memory-safe implementation**: Rust throughout; `unsafe` confined and audited.

### 2. Stability

Component isolation makes crashes survivable: a driver or server crash is a restart
event with state recovery, not a kernel panic. Restart-and-recover is the same
machinery as live update (goal 4), so it is designed once, in the
[driver wire protocol](../design/driver-wire-protocol.md).

### 3. Speed and resource efficiency

**The benchmark is Linux**, on real workloads, on the same hardware. Consequences:

- The kernel boundary is drawn pragmatically (Zircon-style), not ascetically
  (MINIX-style). Hot paths may live in the kernel when isolation can be preserved by
  hardware means.
- Bulk data moves through shared-memory rings, not messages.
- The IPC primitive budget is set by measuring, not by ideology.

### 4. Live upgradability

Everything above the kernel upgrades live, with rollback. The kernel itself upgrades
via near-instant reboot with serialized state handoff (kexec-style): quiesce
components, serialize state, swap kernel, restore. Rollback keeps the old binary and
old state blob until the new version is confirmed healthy.

### 5. Testability — e2e and integration only

No unit tests. Every component is a process with a typed protocol, so the natural
test is: boot a partial system in QEMU with a test-harness component on the other end
of the protocol. Golden transcripts, protocol fuzzing, and full-system scenario tests.

### 6. Modularity

Components are developed, tested, versioned, and shipped independently. The same
component set reconfigures into server, desktop, or mobile products. The monorepo is
structured so any component can be split out without surgery.

## Cross-cutting design principles

Not goals or features — rules that resolve design questions everywhere:

### Multicore-first

There will be no new single-core processors; a single core is just a multicore
system with one core. Therefore: no code that assumes uniprocessor, per-CPU
data structures from the first line of kernel code, no globally serialized
subsystem, and a scaling story that reaches supercomputers — the
message-passing component architecture already leans multikernel
(Barrelfish-style: treat a manycore machine as a distributed system), so
kernel-internal state stays per-CPU and NUMA-aware rather than lock-guarded
and global. Retrofitting this is measured in decades (Linux's BKL removal);
we start there instead.

### Determinism by default, speed by declaration

Concurrent software must be exactly reproducible — crashes replayable from
recorded history, interleavings scriptable in tests — with all instrumentation
**zero-cost when off**. Components escape for speed or real time (busy-poll,
direct timers, RT scheduling, core shielding) only through **declared,
auditable capability grants** that localize the loss of replayability to the
escaping component. See [Determinism](../design/determinism.md). Nothing may
introduce a nondeterministic input that is not kernel-mediated and recordable.

## First-class platform capabilities

These are not priorities that trade off against each other like the list above —
they are feature commitments that must be designed in from the start, because each
one is cheap if native and miserable if retrofitted:

### Hardware virtualization, natively

The kernel already runs a hypervisor (VT-x/EPT, ARM EL2) for the Linux driver VM
and for isolating untrusted driver domains. That machinery is exposed as a
**public, first-class API** (the `VmDomain` kernel object), not kept as internal
plumbing — making Dovenix a type-1 hypervisor by construction. Running guest VMs
(Linux, others) is a supported product feature on desktop and server, KVM-class in
role.

### Containerization, natively

Docker/podman-style workflows are a design target, not a compatibility layer.
Dovenix's architecture is unusually well-shaped for this: the kernel has **no
global namespaces** (all naming is userspace) and **no ambient authority** (all
access is via granted handles), so a "container" is simply a process tree started
with a different namespace map and a bounded resource budget — the isolation
primitive the whole OS is built on, not a bolted-on kernel feature like Linux
namespaces. OCI image and runtime compatibility is the goal so existing
container tooling and registries work.

### Easy porting of existing software (POSIX-first)

Existing Linux/BSD/macOS software must port **without unnecessary change** — the
target is `./configure && make` (or `cargo build`, `cmake`, …) with zero or trivial
patches. This is a deliberate divergence from the Fuchsia philosophy: where the
Zircon-style model conflicts with correct, efficient POSIX (`fork`, signals,
file-backed `mmap`, filesystem semantics, ptys/job control), **POSIX wins at the
boundary** and is allowed to shape kernel primitives. The capability model stays
underneath; per-process namespace maps provide the complete Unix world view. See
the [POSIX strategy](../design/architecture.md#posix-strategy-first-class-ports-without-unnecessary-change).
Porting success is measured by a named package set building unpatched (release
gate from M3). Linux *binary* compatibility remains a non-goal for now — source
compatibility is the commitment.

### Deterministic development and debugging

The multicore-first principle made developer-facing: a deterministic test
scheduler in the standard sync/async primitives (seeded and *scripted*
interleavings), component record/replay via capability interposition,
whole-system deterministic simulation for e2e, and a per-component production
flight recorder. Specified in [Determinism](../design/determinism.md).

### Real-time capability

Priority/deadline scheduling with **priority inheritance across IPC** (the
classic microkernel inversion problem, solved in the kernel object model, not
patched later), core shielding, and locked-memory RT paths. The acceptance
benchmark is pro-audio: sub-millisecond-period full-duplex audio with zero
XRuns under load, as a stock capability
([worked example](../design/determinism.md#511-worked-example-and-acceptance-benchmark-pro-audio-latency)).

### Proper hardware integration

Daily-driver means the machine behaves like a laptop/desktop should: ACPI
(via ACPICA's permissive license branch), sleep/resume (s2idle and S3), battery
status and charge control, thermal management, CPU/device frequency scaling, lid
and power-button policy. Device power transitions ride the same driver-lifecycle
machinery as live update (quiesce/restore), so suspend support is not optional
per-driver heroics — it falls out of protocol conformance.

## Non-goals

- **Formal verification of the kernel.** We take seL4's design lessons, not its proof
  burden. (Revisitable once the design stabilizes.)
- **Supporting legacy hardware.** x86_64 with UEFI and ARM64 only; no BIOS boot, no
  32-bit, nothing without an IOMMU for DMA-capable devices.
- **Binary compatibility with Linux.** POSIX source compatibility is the target;
  a Linux-binary compat layer may come later but is not a design constraint.
- **GPL anywhere outside `drivers/ported/`.** See the
  [Licensing Model](../design/licensing.md).

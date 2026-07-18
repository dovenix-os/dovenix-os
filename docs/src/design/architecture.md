# Architecture Overview

> **Status: draft.** This document sets the system topology and kernel philosophy.
> Details will firm up as the wire protocol and kernel object model are specified.

## Kernel philosophy: Zircon-inspired, hardware-enforced

The Dovenix kernel is a capability-based, "microkernel-ish" kernel in Rust:

- **Small but pragmatic boundary.** Like Zircon and unlike MINIX/seL4, the kernel is
  not ascetic: scheduling, memory objects, IPC primitives, and select hot paths live
  in the kernel. Anything with a large attack surface or third-party code (drivers,
  filesystems, network protocols, POSIX personality) lives in userspace.
- **Everything is a handle.** Processes hold unforgeable capability handles to kernel
  objects (address spaces, threads, memory objects, channels, interrupts, device
  resources). No global namespaces in the kernel; namespaces are a userspace concern.
- **Hardware does the enforcement.** IOMMU for DMA isolation, virtualization
  extensions for untrusted driver domains, MPK/PKS for sub-process compartments,
  CET/PAC/BTI/MTE as baseline hardening.

## Kernel objects (initial set)

| Object | Purpose |
|---|---|
| Process / Thread | Isolation and execution |
| Job | Process-tree container: resource limits, accounting, kill-by-subtree |
| VMO (virtual memory object) | Sharable memory; backing for mappings and rings |
| Channel | Bidirectional message + handle transfer (control plane) |
| Event / Signal | Cheap async notification (data-plane doorbells) |
| Interrupt | Userspace interrupt delivery to drivers |
| IoRegion | MMIO/port access grant, IOMMU-constrained |
| VmDomain | A hardware virtual machine — public hypervisor API; also hosts the Linux driver VM |

## System topology

```
 ┌───────────────────────────────────────────────────────────────┐
 │                        applications (POSIX)                   │
 ├───────────────────────────────────────────────────────────────┤
 │  posixd (personality) │ vfsd │ netd │ ... system servers      │
 ├───────────────────────────────────────────────────────────────┤
 │  devmgr (device manager / driver supervisor)                  │
 │   ├── driver host: virtio-blk   (native, Apache-2.0 OR MIT)   │
 │   ├── driver host: nvme         (native, from spec)           │
 │   ├── driver host: e1000e       (GPL island, ported)          │
 │   └── VmDomain: Linux driver VM (aggregation, unmodified)     │
 ├───────────────────────────────────────────────────────────────┤
 │                      Dovenix kernel (Rust)                    │
 └───────────────────────────────────────────────────────────────┘
```

- **devmgr** is the driver supervisor: enumerates buses, matches drivers, spawns one
  driver host process per driver, grants it exactly the IoRegions/Interrupts/IOMMU
  mappings its device needs, and supervises its lifecycle (crash-restart, live
  update) via the [driver wire protocol](driver-wire-protocol.md).
- **Driver hosts** are isolated processes. Native drivers and ported GPL drivers are
  architecturally identical — the license difference is invisible at runtime, which
  is the point.
- **The Linux driver VM** is an unmodified Linux running in a VmDomain with selected
  physical devices passed through (VFIO-style, IOMMU-isolated). It re-exports those
  devices to Dovenix as virtio devices, which Dovenix consumes with its own native
  virtio drivers. This makes real desktop hardware usable from day one.

## Boot path

```
UEFI (Secure Boot) → Limine → kernel → root server → devmgr → system servers → login
```

Every stage measures the next into the TPM; the verified chain and rollback-protected
A/B system images are specified in a future boot & update document.

## Live update model

- **Components** (drivers, servers): quiesce → serialize state → start new version →
  restore → resume, with the old binary and state blob retained for rollback. This is
  a wire-protocol feature, not per-component heroics.
- **Kernel**: serialized state handoff across a near-instant kernel swap
  (kexec-style). The kernel is the only component whose update implies a (sub-second)
  execution gap.
- **Crash-restart is the same machinery** with an empty state blob and replayed
  configuration — stability (goal 2) and upgradability (goal 4) share one design.

## Virtualization: hypervisor as a public API

The kernel is a type-1 hypervisor by construction — VT-x/EPT and ARM EL2 support
exist from day one because the Linux driver VM and untrusted driver domains require
them. `VmDomain` is therefore a **public kernel object**, not internal plumbing:

- A userspace VMM server (`vmmd`) builds guest VMs from `VmDomain` + VMO-backed
  guest memory + virtio device backends, exactly the way the driver VM is built.
- Guest-facing devices are ordinary Dovenix components: a virtio-blk backend is a
  DWP *client* of a block driver on one side and a virtio device on the other.
- KVM-class role: running Linux (and other) guests is a supported product feature
  for desktop and server, and doubles as the app-compatibility escape hatch.

## Containers: namespaces were never in the kernel to begin with

Dovenix gets Docker/podman-class containerization without a dedicated kernel
subsystem, because the two things Linux had to retrofit are the native primitives
here:

- **Namespacing**: the kernel has no global names — a process sees exactly the
  namespace map (filesystem view, services, network identity) its parent handed it
  at spawn. A container is just a process tree given a different map. There is no
  "escape the namespace" attack class, because there is no global namespace to
  escape into.
- **Resource control**: the `Job` hierarchy carries CPU/memory/handle budgets and
  accounting (cgroup role, but a first-class kernel object).
- **`containerd`-role server**: assembles OCI images into namespace maps, manages
  container lifecycles, and speaks OCI so existing registries and tooling work.
- **Spectrum of isolation**: the same OCI image can run as a process-tree container
  (cheap) or inside a `VmDomain` (Kata-style hard isolation) — the choice is a
  launch flag, since both substrates are native.

## Hardware integration: power as a lifecycle, not a bolt-on

- **ACPI** via ACPICA (permissive Intel license branch), wrapped in a userspace
  `powerd` server: power states, battery, thermal zones, lid/buttons, frequency
  scaling policy.
- **Sleep/resume** (s2idle first, S3 where firmware cooperates): system suspend is
  a coordinated quiesce — `powerd` asks devmgr to drive drivers through the same
  DWP quiesce/restore states used for live update. A driver that passes update
  conformance is suspend-capable by construction; there is no separate,
  perpetually-broken suspend path per driver.
- **Battery/charge management** and thermal policy are `powerd` clients of driver
  classes (`battery`, `thermal` — to be specced alongside `block`).
- The Linux driver VM participates: it receives suspend notifications like any
  other component, using Linux's own PM for the devices it fronts.

## POSIX strategy

POSIX is a personality, not the core: servers speak Dovenix-native protocols;
`posixd` + the C library (mlibc or a fork) translate. This keeps POSIX completeness a
compatibility metric rather than an architectural constraint.

## Open questions

- Exact kernel syscall surface and handle rights model (needs its own doc).
- MPK/PKS compartment policy: which same-address-space boundaries are worth it?
- Scheduler design for the Linux-benchmark goal (needs its own doc).
- Filesystem choice for M3/M4 (port vs. new; candidates: FAT for boot, then a
  permissively-licensed modern FS or a new one).
- Namespace map format and spawn-time handoff ABI (the container substrate — needs
  its own doc before posixd lands).
- Suspend vs. live-update quiesce: same FSM states or a parallel power FSM in DWP?
  (Tracked as DWP open question; current lean: same states, different resume verb.)
- How much of the OCI runtime spec to implement natively vs. adapt via a ported
  runtime shim.

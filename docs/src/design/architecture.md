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
| VMO (virtual memory object) | Sharable memory; backing for mappings and rings |
| Channel | Bidirectional message + handle transfer (control plane) |
| Event / Signal | Cheap async notification (data-plane doorbells) |
| Interrupt | Userspace interrupt delivery to drivers |
| IoRegion | MMIO/port access grant, IOMMU-constrained |
| VmDomain | A hardware virtual machine (hosts the Linux driver VM) |

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

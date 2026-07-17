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

## Non-goals

- **Formal verification of the kernel.** We take seL4's design lessons, not its proof
  burden. (Revisitable once the design stabilizes.)
- **Supporting legacy hardware.** x86_64 with UEFI and ARM64 only; no BIOS boot, no
  32-bit, nothing without an IOMMU for DMA-capable devices.
- **Binary compatibility with Linux.** POSIX source compatibility is the target;
  a Linux-binary compat layer may come later but is not a design constraint.
- **GPL anywhere outside `drivers/ported/`.** See the
  [Licensing Model](../design/licensing.md).

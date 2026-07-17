# Licensing Model

> **Status: policy — locked.** This was decided before the first line of code
> deliberately: license structure cannot be fixed retroactively without chasing every
> contributor.

## Summary

- The entire tree is **`Apache-2.0 OR MIT`** (dual, contributor's choice does not
  apply — contributions are licensed under both).
- The single exception: **`drivers/ported/`** contains drivers derived from Linux
  source, licensed **`GPL-2.0-only`**, isolated as separate processes ("GPL
  islands").
- Nothing GPL is ever a library. Nothing outside `drivers/ported/` may be derived
  from GPL source.

## The reasoning, step by step

### Why permissive at all

Dovenix must be entirely free and permissively licensed (Apache-2.0-class) so it can
be used, embedded, and commercialized without copyleft obligations. That rules out
deriving from GPL codebases for any core component — which is why the kernel is a
fresh Rust implementation (Zircon-*inspired*; Zircon itself is permissively licensed
and serves as reference, while seL4's GPLv2 kernel is off-limits for derivation).

### Why dual `Apache-2.0 OR MIT` and not Apache-2.0 alone

Linux is **GPL-2.0-only**, and the FSF considers Apache-2.0 **incompatible with
GPLv2** (its patent-termination and indemnification terms are "additional
restrictions" under GPLv2). A driver ported from Linux is a GPL-2.0-only derivative
work — and it must link our driver SDK, IPC stubs, and DWP codec crates to function.

If those crates were Apache-2.0-only, the combined driver binary would have no valid
license. Dual-licensing them lets the GPL island link them **under MIT** (which is
GPLv2-compatible), while all other consumers use them under Apache-2.0 and enjoy its
explicit patent grant. This is the standard Rust ecosystem convention, adopted for
exactly this reason.

**Consequence**: every crate a driver could link — in practice, all of `libs/` — MUST
remain `Apache-2.0 OR MIT` and MUST NOT grow GPL-incompatible dependencies.

### Why GPL islands are legally sound

Copyleft extends to *derivative works*, not to programs that merely communicate:

- A ported driver runs as a **separate process** in its own address space,
  communicating exclusively via the versioned
  [driver wire protocol](driver-wire-protocol.md). The rest of the OS is no more a
  derivative of the driver than an application is a derivative of the kernel it
  makes syscalls to.
- Shipping GPL drivers together with the Apache-2.0/MIT system in one image is
  **mere aggregation**, which the GPL explicitly permits.
- Inside the island, linking MIT-licensed crates into the GPL binary is a
  GPL-compatible combination; the combined binary is distributed under GPL-2.0-only
  with full source.

Precedent: Genode ships ported Linux drivers as isolated components under a
different license than the drivers; seL4 systems run non-GPL userlands on a GPLv2
kernel; Linux itself hosts proprietary userspace across the syscall boundary.

### The Linux driver VM: aggregation, not derivation

The driver VM runs an **unmodified** Linux in an isolated VM domain, re-exporting
passed-through devices as virtio. Dovenix consumes them with its own permissive
virtio drivers. Linux here is a separate aggregated program; nothing in Dovenix
derives from it. The VM image is built and shipped as a distinct artifact with its
own (GPL) source offer.

## Operational rules

1. **SPDX headers everywhere.** `Apache-2.0 OR MIT` by default; `GPL-2.0-only`
   inside islands. CI rejects files without a header or with a wrong-scope header.
2. **One directory per island**: `drivers/ported/<name>/` with its own `LICENSE`,
   plus a `PROVENANCE.md` recording upstream repo, kernel version, and commit hash.
3. **Clean-room discipline for the shim.** The Linux-API compatibility layer
   (`libs/linux-shim/`) is written from documentation and driver-facing API
   surfaces only — never while reading GPL implementation source — and stays
   `Apache-2.0 OR MIT`.
4. **Dependency policy**: no GPL/LGPL/AGPL/MPL dependencies outside islands;
   `cargo deny` (or equivalent) enforces this in CI from the first crate.
5. **Firmware blobs** (Wi-Fi, GPU) are redistributable-non-free vendor artifacts,
   shipped in a clearly separated `firmware` package, never linked, with their
   redistribution terms recorded. (A fully blob-free configuration remains possible
   with reduced hardware support.)

## Not legal advice

This model follows well-established community understanding (Linux syscall
boundary, seL4 userland statements, GPLv2 §2 "mere aggregation"). Before any
commercial distribution or relicensing event, have counsel bless the
process-boundary + dual-licensed-SDK structure.

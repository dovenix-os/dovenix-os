# Introduction

Dovenix is a modern, secure, POSIX-compatible operating system built in Rust.

The one-sentence pitch: **microkernel-grade isolation, enforced by modern hardware
security features instead of paid for with IPC everywhere, benchmarked against Linux
on speed.**

Dovenix follows the design philosophy pioneered by Fuchsia's Zircon — a
[pragmatic, capability-based "microkernel-ish" kernel](design/architecture.md#kernel-philosophy-zircon-inspired-hardware-enforced)
— but is implemented fresh in Rust under
permissive licenses (`Apache-2.0 OR MIT`), drawing on seL4's security literature,
and on Redox and Managarm as existence proofs that a small team can ship a
Rust-based POSIX-capable OS.

One deliberate departure from Fuchsia: **POSIX is first-class, not an emulation
detail.** Existing Linux/BSD/macOS software must port without unnecessary change,
and where Zircon's model conflicts with that (`fork`, signals, POSIX filesystem
semantics),
[POSIX wins at the boundary](design/architecture.md#posix-strategy-first-class-ports-without-unnecessary-change)
— the capability model stays underneath.

## Where the project is today

Design phase. The documents in this book are the current output of the project, in
this order of importance:

1. [Driver Wire Protocol](design/driver-wire-protocol.md) — the load-bearing
   interface: license boundary, fault isolation, live update, and testability all
   depend on it.
2. [Architecture Overview](design/architecture.md) — the kernel model and system
   topology.
3. [Licensing Model](design/licensing.md) — why the tree is `Apache-2.0 OR MIT`
   with isolated GPL islands, and the rules that keep it that way.

## Reading guide

If you are evaluating the project, read the [Goals](vision/goals.md) first — every
design decision in this book traces back to that priority-ordered list, and trade-offs
are resolved by priority order, not taste.

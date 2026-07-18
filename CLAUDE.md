# Dovenix OS — agent guide

Dovenix is a from-scratch OS in Rust: Zircon-inspired capability kernel,
userspace drivers/servers, POSIX-first compatibility. Pre-alpha; specs lead code.

## Read before working

1. `docs/src/vision/goals.md` — priority-ordered goals; trade-offs resolve by
   this order, not taste.
2. The design doc for whatever you're touching (`docs/src/design/`). The book
   is **normative**: if code or plan disagrees with it, the book wins — change
   the book first (same PR) or conform.
3. The **active plan** in `plans/` (exactly one milestone plan has
   `status: active`) — it sequences the work and tracks status. Update it in
   the same commit that changes reality. See `plans/README.md`.

## Hard rules

- **Licensing**: everything is `Apache-2.0 OR MIT` with SPDX headers on every
  source file. GPL-derived code goes only in `drivers/ported/<name>/`
  (`GPL-2.0-only`), always a separate process, never a library. Never copy
  from or reference GPL source anywhere else. Details: CONTRIBUTING.md and
  `docs/src/design/licensing.md`.
- **No unit tests.** Tests are e2e/integration only, living in `tests/`,
  asserting observable behavior of booted (partial) systems or host tools.
  Don't add `#[cfg(test)]` modules; add harness scenarios instead.
- **Dependency direction**: components depend only on `libs/`, never on each
  other's internals. `kernel/` depends on nothing in-tree.
- **Wire protocols are contracts**: any DWP or class change requires the
  version bump + doc update in the same change
  (`docs/src/design/driver-wire-protocol.md` §7).
- **Multicore-first**: no code that assumes uniprocessor; kernel state is
  per-CPU, never guarded by new global locks.
- **Determinism invariants** (`docs/src/design/determinism.md` §4): never add
  a nondeterministic input that isn't kernel-mediated and recordable; system
  components use only `libs/` sync/async primitives (they carry the schedule
  points); escape hatches are declared capability grants, never smuggled.

## Building

Nothing builds yet (design phase). `cargo xtask` will be the entry point for
build/image/run/e2e once M1 Phase 1 lands; keep it that way — no bespoke
scripts scattered around.

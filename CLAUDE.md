# Dovenix OS — agent guide

Dovenix is a from-scratch OS in Rust: Zircon-inspired capability kernel,
userspace drivers/servers, POSIX-first compatibility. Pre-alpha; specs lead code.

## Read before working

1. `docs/src/vision/goals.md` — priority-ordered goals; trade-offs resolve by
   this order, not taste.
2. `docs/src/design/principles.md` — the **index of load-bearing principles**,
   one line each with links. Skim it before any design-adjacent decision; if
   an entry bears on your decision, follow its link first. If your change
   introduces or alters a load-bearing principle, update the index in the
   same PR.
3. The design doc for whatever you're touching (`docs/src/design/`). The book
   is **normative**: if code or plan disagrees with it, the book wins — change
   the book first (same PR) or conform.
4. The **active plan** in `plans/` (exactly one milestone plan has
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

## Development workflow (mandatory — this OS is developed mostly by AI agents)

The loop below is not optional and has no fast path. Human checkpoints are the
product's safety mechanism; skipping one is never a favor.

1. **Plan first, approved by a human.** No feature or milestone implementation
   starts without a phased plan in `plans/` whose frontmatter carries
   `approved-by` filled in by a human. `status: active` without approval is
   invalid. Propose the plan, then stop and wait.
2. **Design goes in the book.** The book (`docs/`) is the canonical
   description/definition of the system. Any design decision or hard spec that
   emerges during planning or implementation lands as a book change (same PR),
   subject to the same human approval. Plans sequence design; they never
   contain it.
3. **Phases end in something a human can verify.** Each phase's outcome must be
   directly checkable — an e2e scenario that runs green, an observable demo, a
   reviewable spec. "Internal progress" that a human can't exercise is not a
   phase boundary.
4. **Phase completion protocol**, in order, before asking for review:
   a. **Audit** the phase's work, in this order: first **reliability** (no
      silent failures, errors propagated or handled deliberately, resources
      cleaned up on all paths), **Rust-native idioms** (clippy-clean, no
      C-brain patterns, ownership expresses the design), and **performance
      mistakes** (unnecessary copies/clones and allocations, needless heap
      round-trips, work on hot paths that belongs at setup time, accidental
      O(n²), lock scope wider than needed — goal 3 is benchmarked against
      Linux, so casual waste is a bug), applying fixes; then
      **security last** (unsafe blocks justified and minimal, all external
      input validated, capability scope not widened), so it reviews the code
      as it will actually ship — a security pass is invalidated by any fix
      applied after it. If a security finding forces changes, re-run security
      on the result. Fix what you find or report what you can't.
   b. **Update all relevant documentation** — book docs, plan status/deviations,
      READMEs, and **docstrings** on public items — to match what was actually
      built, and **confirm the documentation updates with the user**.
   c. **Pause for human review.** Do not start the next phase on your own.
5. **Committing**: no implementation commit until step 4b's documentation is
   updated and confirmed. (Docs-only and plan-only commits don't need the
   audit, but design changes still need approval per rule 2.)

## Building

Nothing builds yet (design phase). `cargo xtask` will be the entry point for
build/image/run/e2e once M1 Phase 1 lands; keep it that way — no bespoke
scripts scattered around.

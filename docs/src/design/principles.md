# Load-Bearing Principles

An **index, not a source**: every principle that resolves design questions
across the system, stated in one line, linking to where it is actually
specified. If you are about to make a design-adjacent decision, skim this page
first — if a principle here bears on your decision, follow its link before
proceeding.

**Maintenance rule**: any change that introduces, strengthens, or weakens a
load-bearing principle MUST update this index in the same PR. If a principle's
one-liner and its source doc disagree, the source doc wins (and the index is a
bug to fix).

## Authority and process

- **The book is canonical; when anything disagrees with it, the book wins.**
  Change the book first or conform. → [Repo layout](../dev/repo-layout.md),
  `plans/README.md`
- **Trade-offs resolve by the priority-ordered goals, not taste.**
  → [Goals](../vision/goals.md)
- **Humans gate every plan and every phase.** Approval before implementation;
  audit (reliability, idioms, performance, then security last) and confirmed
  docs before review. → `CLAUDE.md`, `plans/README.md`

## Security and isolation

- **No ambient authority — a component's maximum reach is what was explicitly
  granted.** Handles are the only authority; for drivers, all device authority
  arrives at `BIND`. → [Architecture](architecture.md),
  [DWP §6.1](driver-wire-protocol.md)
- **Authority is what a process can reach, not what it holds: system servers
  are never ambient-authority oracles.** Whatever an endpoint hands out is
  derived from the requesting connection's namespace map, so a spawned
  subtree's reachable authority stays a subset of its spawner's at every
  delegation depth (this is what makes nested agents safe).
  → [Architecture: agents](architecture.md#agents-the-container-substrate-plus-a-grant-model)
- **IOMMU is mandatory for anything that touches DMA, with no bypass; its cost
  is defeated by the registration model (map at queue creation, never
  per-operation).** → [Architecture: DMA isolation](architecture.md#dma-isolation-and-the-registration-model)
- **Isolation is a cost-ordered ladder — pick the cheapest rung that meets the
  threat**: same-compartment → MPK/PKS window (bug containment, not a boundary
  against arbitrary code execution unless gadget-controlled) → process →
  VmDomain. → [Architecture: compartments](architecture.md#intra-address-space-compartments-mpkpks-and-the-isolation-ladder)
- **CPU-level mitigations are baseline, not opt-in**: every binary is built
  with them, the kernel enforces whatever the silicon offers, and they harden
  isolation rungs but never substitute for one.
  → [Architecture: CPU-level mitigations](architecture.md#cpu-level-mitigations-the-baseline-hardening-set)
- **Everything crossing a trust or fault boundary is adversarial** — wire
  input, ring contents, indices: validate at consumption, degrade to errors,
  never crash or hang. → [DWP §8.5, §11](driver-wire-protocol.md)
- **GPL code exists only as isolated processes in `drivers/ported/`; every
  linkable library stays GPLv2-compatible (`Apache-2.0 OR MIT`).**
  → [Licensing](licensing.md)

## Concurrency and determinism

- **Multicore-first: no code assumes uniprocessor; kernel state is per-CPU,
  never behind new global locks.** → [Goals](../vision/goals.md),
  [Architecture](architecture.md)
- **Determinism by default, speed by declaration: nondeterministic inputs must
  be kernel-mediated and recordable; escapes are declared capability grants
  with local (never global) cost to replayability.**
  → [Determinism](determinism.md)
- **All instrumentation is zero-cost when off** — anything that can't prove it
  in measurement doesn't ship in the default build.
  → [Determinism §5.2](determinism.md)
- **RT is designed into the object model**: priority inheritance across IPC
  chains, budgets not just priorities, core shielding; verified by
  measurement, not replay. → [Determinism §5.1](determinism.md)

## Protocols and lifecycle

- **Protocols are contracts**: any wire-visible change carries a version bump
  and doc update in the same change; minor versions only append.
  → [DWP §7](driver-wire-protocol.md)
- **Crash-restart is the degenerate live update** — one
  quiesce/serialize/restore machinery serves stability, upgrades, and
  (via powerd) suspend. → [DWP §6](driver-wire-protocol.md),
  [Architecture](architecture.md)
- **Correctness never depends on peer volatile state surviving**: every
  operation declares its retry class after `ERR_PEER_RESET`; clients hold the
  recovery discipline (re-flush, replay config, resync state).
  → [block §6](classes/block.md), [net §6](classes/net.md),
  [input §6](classes/input.md), [console §6](classes/console.md)
- **Config-plane messages are idempotent absolute state** (replace, never
  increment) so restart recovery is blind replay. → [net §3.3](classes/net.md)
- **Loss, where permitted, is bounded and announced — never silent; completions
  are honest** ("handed off", never "delivered"). → [net §5](classes/net.md),
  [console §5](classes/console.md), [input §5](classes/input.md)

## Compatibility

- **POSIX wins at the boundary; capability-native underneath.** POSIX is
  allowed to shape kernel primitives (fork, signals, mmap); ported software
  builds unpatched as a release gate.
  → [Architecture: POSIX strategy](architecture.md)
- **Per-process namespace maps provide the entire Unix world view — and are
  the container substrate.** No global names anywhere.
  → [Architecture](architecture.md)

## Structure and testing

- **Components depend only on `libs/`, never on each other's internals;
  `kernel/` depends on nothing in-tree.** → [Repo layout](../dev/repo-layout.md)
- **No unit tests: every component is verified end-to-end through its real
  protocol; passing conformance is the definition of "done".**
  → [Testing](../dev/testing.md), [DWP §12](driver-wire-protocol.md)
- **System components use only `libs/` sync/async primitives** — they carry
  the schedule points that make L1 determinism work.
  → [Determinism §4.2](determinism.md)

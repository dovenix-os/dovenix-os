# Determinism, Record/Replay, and Escape Hatches

> **Status: draft.** Design direction and day-one invariants are settled;
> mechanism details firm up as the layers land (schedule points with the M2
> async runtime, simulation harness later).

## 1. Why

Dovenix is [multicore-first](../vision/goals.md): concurrency is the normal
case, so the *development experience* of concurrent software is a
first-class concern. The commitments:

- A crash — even in heavily multithreaded code — can be reproduced from its
  recorded history, exactly.
- Concurrency tests can pin down interleavings at the most granular level and
  reproduce any failure from a seed.
- All of this is **opt-in and zero-cost when off**: production default is full
  speed, and speed-critical/real-time components have declared escape hatches
  (§5).

This serves stability (goal 2: heisenbugs become replayable), testability
(goal 5: deterministic e2e), and the developer-experience side of modularity.

## 2. Why this is tractable here

Two facts, both already load-bearing in the architecture, make exact replay
affordable when almost everywhere else it is not:

1. **Safe Rust is data-race-free by construction.** In a DRF program, threads
   interact only through explicit synchronization (locks, atomics, channels).
   Exact replay therefore requires recording only the **order of
   synchronization events** plus external inputs — not memory traffic. This
   is the difference between ~100× overhead and a few percent.
2. **A component is a deterministic function of its message history.** All
   nondeterminism enters through enumerable kernel interfaces: channel
   messages, event wakeups, ring index transitions, time queries, RNG. No
   ambient authority means no hidden input channels. (DWP golden transcripts
   are the primitive form of this; this doc is that idea taken to its end.)

## 3. The four layers

Cheapest to most expensive; independently useful; all seed- or
log-reproducible.

### L1 — Deterministic test scheduler (in-process)

Every lock, atomic wrapper, channel op, and `await` point in `libs/` sync
primitives and the async runtime is a **schedule point**. In test mode, a
controlling scheduler owns every interleaving decision:

- randomized exploration with seed reproduction (shuttle-style), and
  exhaustive exploration for small state spaces (loom-style);
- **scripted schedules**: tests state "thread A runs to this await; B acquires
  the lock; A resumes" to construct a precise state — interleavings become
  test fixtures, not luck.

Because the primitives are *ours* (and ported POSIX software reaches them
through mlibc's pthread layer), this works transparently for native and
ported code alike.

### L2 — Component record/replay (interposition)

The capability system gives recording for free: devmgr (or a debug tool) can
interpose on any component's channels and event sources **without the
component's knowledge or cooperation**, recording inbound messages, wakeups,
and ring transitions with logical timestamps. Replay runs the component solo
in a harness, off-system, bit-exactly (with L1 hooks active for in-process
sync order when maximum granularity is needed). This is the
"driver crashed on someone's machine — replay its exact history at my desk"
story.

### L3 — Whole-system deterministic simulation

FoundationDB/TigerBeetle-lineage: the entire OS runs against virtual time
with simulated cores multiplexed onto host threads — wall-clock parallelism
is sacrificed, concurrency *semantics* are kept, one seed reproduces an
entire system execution including injected faults (crashes, delays, torn
messages). This is the stated end-state of the e2e harness (testing
strategy): the multicore-first principle read backwards — a multicore system
simulated on one core is still the same system.

### L4 — Production flight recorder (the debug mode)

A bounded per-component ring of sync-order + IPC events, **enabled per
component** via L2 interposition plus an in-process sync log: fly one suspect
driver in record mode on an otherwise full-speed system. A crash ships with
its replayable recent history. Overhead is paid only by the component under
observation.

## 4. Day-one invariants (cannot be retrofitted)

1. **Nondeterminism stays enumerable.** No vDSO-style time reads that bypass
   kernel mediation in the default ABI; unmediated `RDTSC`/counter access is
   an escape hatch (§5), trapped otherwise in recorded modes (`CR4.TSD`);
   shared memory with outside writers exists only as declared objects (rings,
   named shared VMOs); ASLR and other intentional randomness are seedable and
   suspendable in debug modes. Nothing may add a nondeterministic input
   without it being kernel-mediated and recordable. (CLAUDE.md hard rule.)
2. **Primitives are hook-ready and zero-cost when off.** `libs/` ships *the*
   mutex, atomics wrappers, channels, and executor with schedule points that
   compile to nothing in release configuration. No third-party sync
   primitives in system components.
3. **Logical time is first-class in protocols.** Messages and events carry
   logical timestamps with explicit clock domains (input's
   interrupt-time stamps are the template), so recorded histories are
   self-ordering across components.

## 5. Escape hatches: speed and real time

Determinism is **compositional and per-component; escape is declared, never
smuggled**. A component may hold *nondeterminism capabilities*, granted like
any other capability (manifest-declared, devmgr-granted, auditable):

| Escape hatch | Grants | Cost to observability |
|---|---|---|
| `time.direct` | Unmediated cycle counters / direct time reads | Time values unrecordable; component replay becomes input-boundary-approximate |
| `mem.shared-fast` | Shared memory with another component outside ring discipline | Cross-component sync order unrecorded on that surface |
| `sched.poll` | Busy-poll rings, no doorbell waits (DPDK/io_uring-class data paths — the ring design already supports poll mode) | Wakeup ordering unrecorded on that queue |
| `sched.rt` | Real-time scheduling (§5.1) | RT paths verified by measurement, not replay |

Consequences are local: a component that takes escape hatches forfeits exact
replay *for itself* (L2 still records its message boundaries), and the rest
of the system remains fully recordable. The set of granted escapes is visible
system state — performance archaeology starts from a list, not a hunch.

### 5.1 Real time

Real-time is a scheduling commitment, not only a determinism escape:

- **Scheduling classes**: priority/deadline classes alongside the fair class;
  design informed by seL4's MCS work (budgets, not just priorities).
- **Priority inheritance across IPC** — the classic microkernel RT failure is
  priority inversion through a shared server; the channel/scheduler design
  must propagate effective priority along request chains. This is a kernel
  object-model requirement, flagged now so the M1 handle/channel design
  doesn't preclude it.
- **Core shielding**: on a multicore-first OS the natural RT (and HPC) model
  is *spatial*: dedicate cores — tickless, no unrelated threads, steered
  interrupts, locked memory — to an RT or throughput-critical component.
  Multicore-first makes this cheap; it is also the supercomputer story
  (kernel-bypass data paths + shielded cores = HPC-grade behavior from the
  same primitives).

### 5.1.1 Worked example and acceptance benchmark: pro-audio latency

Extremely low-latency audio is the canonical RT target for a daily-driver OS —
the most demanding consumer real-time workload, and unforgiving in a
microkernel design because the signal path crosses components. A 32-sample
period at 48 kHz is a **~0.7 ms hard deadline, every period, or it's audible**.
The path exercises every RT mechanism above at once:

- audio server and client DSP threads run `sched.rt` with locked memory
  (a page fault mid-period is a missed deadline);
- the device period interrupt propagates as a priority-inheriting wakeup chain
  (driver → audio server → client) — multi-hop inversion is exactly the §5.1
  IPC-priority problem;
- buffers move through DWP-style rings shared along the chain (zero copies on
  the period path), with `sched.poll` as the sub-period option for extreme
  cases;
- a shielded core is the escalation for hard sessions (recording studio mode).

**Acceptance benchmark (M4-class hardware): a 48 kHz / 32-sample full-duplex
round-trip running for hours with zero XRuns under system load** — JACK-grade
behavior as a stock capability, not a patched-kernel specialty. A future
`audio` DWP class (isochronous streaming with period timing contracts —
unlike anything in the current class set) gets specced when this lands; it
will also be the template for other isochronous devices (USB audio, video
capture).

Every hook in this document must be **zero-overhead when disabled** —
release-mode Dovenix with no recording active concedes nothing to the Linux
benchmark. Any layer that can't meet that bar in measurement doesn't ship in
the default build.

## 6. Mode summary

| Mode | What runs | Guarantee | Overhead |
|---|---|---|---|
| release | everything | escape hatches per grants; boundaries recordable on demand | none |
| record (per-component) | L2/L4 on chosen components | exact replay of those components | few %, local |
| test | L1 scheduler in-process | seed/script-exact interleavings | debug-build only |
| sim | L3 whole system | seed-exact system execution incl. faults | no wall-clock parallelism (CI only) |

## 7. Open questions

- Sync-order log format and its relationship to DWP transcripts (one format?).
- Atomics recording granularity: per-op logging vs. Kendo-style deterministic
  logical clocks — measure both on the M2 runtime.
- How `time.direct` interacts with L3 simulation (virtualized TSC offsets vs.
  denying the grant in sim).
- Priority-inheritance mechanics across multi-hop server chains (needs the
  syscall/object doc from M1 Phase 4).
- Whether L4's flight-recorder ring lives in the component or the supervisor
  (crash-survivability argues supervisor-side).

# Testing Strategy

> **Status: draft.** The stance is settled
> ([goal 5](../vision/goals.md#5-testability--e2e-and-integration-only)); the
> harness design will firm up at milestone M1.

## The stance: e2e and integration only, no unit tests

Every Dovenix component is a process with a typed protocol. The natural unit of
testing is therefore *a component behind its real protocol*, not a function behind a
mock. Unit tests are rejected deliberately: they test the mock's assumptions, drift
from real behavior, and add refactoring drag without adding boot confidence.

## The three layers

### 1. Protocol conformance (per component)

Boot a minimal system in QEMU containing: kernel, devmgr, the component under test,
and a **harness component** on the other end of its protocol. The harness asserts:

- [lifecycle FSM](../design/driver-wire-protocol.md#6-lifecycle-state-machine)
  correctness (HELLO/BIND/START, quiesce under load, restore fidelity)
- version negotiation, including rejection paths
- crash-restart semantics: kill the component mid-operation, assert every in-flight
  transaction resolves per its declared
  [retry class](../design/driver-wire-protocol.md#63-crash-restart)
- for drivers: the class conformance suite (see the
  [wire protocol](../design/driver-wire-protocol.md), §12)

Passing conformance is the definition of "done" for a component.

### 2. Scenario tests (partial systems)

Compose real components into partial systems and drive workloads: vfsd + virtio-blk
doing a fsync storm; netd + virtio-net under packet loss; live-updating a driver
while a client hammers it. Assertions are on externally observable behavior and
recorded transcripts — never on internals.

### 3. Full-system tests

Boot the whole OS in QEMU (later: on real-hardware runners), run POSIX workloads,
upgrade live, roll back, compare performance counters against the Linux baseline on
identical virtual hardware. These are the CI gate for releases and the home of the
performance benchmark suite (goal 3).

## Cross-cutting machinery

- **Golden transcripts**: recorded protocol sessions replayed against new builds;
  any wire-visible change must be an intentional, versioned one.
- **Structured fuzzing**: codecs and ring handling fuzzed from the same schema that
  generates them; drivers fuzzed through their service channels.
- **Fault injection as a first-class harness feature**: process kills, deadline
  overruns, malformed messages, hostile ring states.
- **Determinism**: harnesses control time and randomness sources so failures
  replay from a seed. This is not best-effort — it is the harness's end state:
  whole-system deterministic simulation (virtual time, multiplexed simulated
  cores, seeded fault injection), with scripted-interleaving tests at the
  component level via the schedule points built into `libs/` primitives. The
  full layer stack is specified in [Determinism](../design/determinism.md).
  RT/escape-hatch paths are the exception: they are verified by measurement
  (latency/deadline benchmarks under load), not by replay.

## Why this works here (and doesn't on a monolith)

On a monolithic kernel, "integration test" means "boot everything"; the granularity
gap forces unit tests. Dovenix's component architecture makes integration tests
*cheap and precise*: a partial system with two components boots in well under a
second in QEMU. The architecture is what makes the no-unit-test stance viable —
this is goal 6 (modularity) paying for goal 5 (testability).

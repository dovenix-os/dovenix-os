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
- **Hardware does the enforcement.**
  [IOMMU for DMA isolation](#dma-isolation-and-the-registration-model),
  [virtualization extensions for untrusted driver domains](#virtualization-hypervisor-as-a-public-api),
  [MPK/PKS for sub-process compartments](#intra-address-space-compartments-mpkpks-and-the-isolation-ladder),
  [CET/PAC/BTI/MTE as baseline hardening](#cpu-level-mitigations-the-baseline-hardening-set).
- **Multicore-first.** Per-CPU kernel state from the first commit, no global
  locks guarding subsystems, NUMA-aware allocation/scheduling, and a
  multikernel-leaning scaling posture (per-core state + messages, matching the
  component architecture) that targets everything from one core to
  supercomputers. Scheduling includes RT classes with priority inheritance
  propagated along IPC chains — a kernel object-model requirement, designed in
  now (see [Determinism §5.1](determinism.md)).
- **Deterministic by default, fast by declaration.** All nondeterminism enters
  through enumerable, recordable kernel interfaces; components take declared
  capability escapes (busy-poll, direct timers, RT, core shielding) for speed.
  Full design: [Determinism, Record/Replay, and Escape Hatches](determinism.md).

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
| Grant | Revocation domain: handles tethered to it die when it is revoked — [capability model](capabilities.md#grants-revocation-as-an-object-not-a-tree-walk) |
| VmDomain | A hardware virtual machine — [public hypervisor API](#virtualization-hypervisor-as-a-public-api); also hosts the Linux driver VM |

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
- **Driver hosts** are isolated processes. Native drivers and
  [ported GPL drivers](licensing.md#why-gpl-islands-are-legally-sound) are
  architecturally identical — the license difference is invisible at runtime, which
  is the point.
- **The Linux driver VM** is an
  [unmodified Linux](licensing.md#the-linux-driver-vm-aggregation-not-derivation)
  running in a VmDomain with selected physical devices passed through (VFIO-style,
  IOMMU-isolated). It re-exports those
  devices to Dovenix as virtio devices, which Dovenix consumes with its own native
  virtio drivers. This makes real desktop hardware usable from day one.

## DMA isolation and the registration model

**Why the IOMMU is mandatory.** DMA bypasses the CPU's MMU: an unconstrained
device can read or write *any* physical memory. Without IOMMU enforcement,
driver process isolation is fiction — a compromised driver simply programs its
device to DMA over kernel memory, and no CPU-side mechanism ever sees it
(malicious peripherals — Thunderbolt/PCILeech-class attacks — do the same
without any driver compromise). The IOMMU (VT-d/AMD-Vi/SMMU) is an MMU for
devices: per-device page tables on every DMA. With it, a
[`BIND` grant](driver-wire-protocol.md#61-bring-up) is hardware-enforced on
**both** sides of a driver — the MMU bounds the process, the IOMMU bounds its
device to exactly the granted data VMOs. Three pillars of
this book stand on that: driver crash/compromise containment, the GPL-island
trust story, and the safety of Linux-driver-VM device passthrough. Hence the
non-goal: no support for DMA-capable hardware without an IOMMU.

**The cost model — and why we don't pay the famous part.** IOMMU overhead has
two distinct components:

1. *Translation*: IOTLB hits are free; misses add a page-table walk (hundreds
   of ns) to that DMA. Mitigated with large-page-backed, contiguous VMOs.
2. *Map/unmap*: creating mappings writes page tables; destroying them forces
   IOTLB invalidation — slow and synchronizing. The traditional streaming-DMA
   model (map before each I/O, unmap after) pays this **per operation**; it is
   why high-rate NICs historically ran `iommu=off` or "lazy" invalidation
   modes that trade a security window for throughput.

Dovenix avoids cost 2 **structurally**: data VMOs are IOMMU-mapped **once, at
queue creation** ([DWP §8.1](driver-wire-protocol.md#81-queue-anatomy)),
descriptors reference offsets within those
long-lived mappings, and no hot path ever touches IOMMU state — teardown and
rebind happen at session end or under quiesce, off the fast path. This is the
registration model RDMA, DPDK, and io_uring's registered buffers converged on;
the DWP queue design requires declared, long-lived shared memory anyway (as
does [determinism](determinism.md) §4.1 — the constraints rhyme), so the fast
mapping model falls out of the architecture rather than fighting it.

**No bypass exists.** Nondeterminism escape hatches (`mem.shared-fast`,
`sched.poll` — [Determinism §5](determinism.md#5-escape-hatches-speed-and-real-time))
relax *observability*, never DMA containment; there is no
`iommu=off`. Security is goal 1, speed is goal 3 — the answer to IOMMU cost is
the registration model, not a hole in the foundation.

**Worked check — the pro-audio deadline.** Audio is the easiest DMA client in
the system despite the hardest deadline: ~384 KB/s bandwidth, one tiny ring
buffer cyclically re-DMA'd for the whole session — mapped once, one
permanently hot IOTLB entry (or a single 2 MB page). Even a worst-case IOTLB
walk is ≲0.1% of a 0.7 ms period budget. The IOMMU is a non-factor for the
zero-XRun benchmark *because of* the registration model; the real deadline
risks (wakeup latency, priority inversion, page faults, SMIs) are addressed in
[Determinism §5.1](determinism.md). The residual case where IOMMU cost stays
measurable even with registration — extreme small-packet networking at
100G-class rates (IOTLB pressure) — is an M4+ tuning problem (page sizes,
IOVA layout), not an architectural one.

## Intra-address-space compartments: MPK/PKS and the isolation ladder

**The mechanism.** Changing page permissions normally means `mprotect`-class
work: a syscall, page-table edits, TLB shootdowns — microseconds, worse on
many cores. Memory Protection Keys add indirection: each PTE carries a 4-bit
key (16 per address space, assigned once); the *effective* permissions per key
live in a **per-thread register** — PKRU in user mode (Intel MPK/PKU, modern
AMD too), the `IA32_PKRS` MSR in kernel mode (PKS); ARM64's equivalent is POE
(permission overlays). Writing that register is unprivileged and costs
~10–20 cycles: no syscall, no TLB effect, per-thread. Tag a sensitive region
with a key, default every thread to "no access", and open a nanosecond
*window* (write register → touch → write register) only in the code paths
meant to touch it.

**The honest limit.** Because the register write is unprivileged, code with
arbitrary execution inside the process can simply re-open the key. MPK alone
is therefore **not** a security boundary against a compromised compartment —
it becomes one only with control-flow integrity plus `WRPKRU`-gadget
scrubbing (ERIM/Hodor lineage;
[CET/BTI help](#cpu-level-mitigations-the-baseline-hardening-set)), which is
opt-in machinery, not
the default assumption. What it *is*, at near-zero goal-3 cost, is a
**bug-containment and exploit-hardening boundary**: stray writes, stale
pointers, slips in `unsafe` blocks fault instantly instead of silently
corrupting state. That is a goal-2 tool with goal-1 defense-in-depth value —
which is exactly the register in which the kernel-philosophy bullet cites it.

**The isolation ladder.** Dovenix has four cost-ordered isolation rungs; the
design rule is *pick the cheapest rung that meets the threat*:

| Rung | Cost | Boundary against |
|---|---|---|
| 1. Same compartment | free | nothing |
| 2. MPK/PKS compartment | ~ns window flips | bugs and stray writes; exploits only with gadget control |
| 3. Process | context switch + IPC | compromised code (MMU + capabilities) |
| 4. VmDomain | VM exit costs | compromised kernel-adjacent code (EPT/EL2 + IOMMU) |

Drivers and servers sit on rung 3, untrusted driver domains and guests on
rung 4. Rung 2 exists for boundaries *inside* one component, where a process
split would cost IPC on a hot path.

**Intended uses.** Userspace (MPK): a server keeps its crown jewels —
`vfsd`'s cache metadata, key material in a crypto service, the flight-recorder
ring ([determinism L4](determinism.md#l4--production-flight-recorder-the-debug-mode)) — writable only inside tiny windows, so 99% of its own
code can't corrupt them; a driver host can make its ring-handling module the
only writer of ring memory. Kernel (PKS): page tables, IOMMU tables, and
handle tables stay write-disabled outside the narrow paths meant to mutate
them, so a stray write in the kernel's `unsafe` core becomes an immediate
fault, not an escalation primitive. Keys are scarce (16): a budgeted resource
for a few high-value regions, not sprinkled everywhere — the compartment
policy (which regions, key budget, whether rung 2 is ever promoted to a true
security boundary via gadget scrubbing) gets its own design doc when the
first servers exist (M2/M3; see open questions).

## CPU-level mitigations: the baseline hardening set

**What they are.** Modern cores ship a set of exploit-chain breakers: each one
turns a step of the classic exploitation playbook into a hardware fault, at
near-zero runtime cost. [Goal 1](../vision/goals.md#1-security) makes them
baseline. The set, by the attack step it breaks:

| Attack step broken | x86 | ARM64 |
|---|---|---|
| Kernel executes code in user pages | SMEP | PXN |
| Kernel dereferences an unvalidated user pointer | SMAP | PAN |
| Overwrite a return address (stack smash → ROP) | CET shadow stacks | PAC |
| Indirect branch into a gadget (JOP; `WRPKRU` abuse) | CET IBT | BTI |
| Use-after-free / heap out-of-bounds | — | MTE |

**Privilege boundary (SMEP/SMAP, PXN/PAN).** SMEP and PXN make user pages
never-executable in kernel mode, killing the historic kernel exploit: corrupt
one kernel function pointer, aim it at shellcode the attacker staged in their
own process. SMAP and PAN close the data-access version: kernel-mode reads and
writes of user pages fault unless the kernel explicitly opens a window
(`stac`/`clac` on x86, the PAN bit on ARM64) around the access. The design
consequence sits in the kernel: user memory is reached **only** through
dedicated copy-in/copy-out primitives that validate then copy inside that
window — never dereferenced in place — so an unvalidated user pointer touched
anywhere else in the kernel faults instantly instead of becoming an arbitrary
read/write primitive.

**Backward-edge control flow (CET shadow stacks, PAC).** A CET shadow stack
is a second, hardware-maintained stack that holds only return addresses and
rejects ordinary stores; `ret` pops both stacks and faults on mismatch, so a
smashed return address aborts instead of starting a ROP chain. ARM64 reaches
the same end by cryptography instead of duplication: PAC (pointer
authentication) signs a pointer into its unused upper bits with a per-context
key on the way in and authenticates it before use, so a corrupted or forged
return address fails authentication and faults. PAC generalizes beyond return
addresses — function and data pointers can carry signatures too.

**Forward-edge control flow (IBT, BTI).** Every legitimate indirect-branch
target starts with a landing-pad instruction (`endbr64` / `bti`); an indirect
call or jump landing anywhere else faults. This kills jump-oriented
programming — and it is the enabling half of
[promoting an MPK compartment to a security boundary](#intra-address-space-compartments-mpkpks-and-the-isolation-ladder):
scrubbing `WRPKRU` gadgets only means something when an attacker cannot land
mid-instruction-stream to synthesize new ones.

**Memory tagging (MTE).** Physical memory carries a 4-bit tag per 16-byte
granule; pointers carry a matching tag in their top byte; the allocator
re-colors memory on allocate and free. A load or store whose pointer tag
mismatches the memory's tag faults — synchronously (precise, costlier) or
asynchronously (near-free, batched) — catching use-after-free and
out-of-bounds at hardware speed. Detection is probabilistic (15/16 per
access), which is ample for catching bugs and breaking exploit reliability.

**Where each is applied.**

- *Enabled by the kernel at boot*, per feature, wherever the silicon has it:
  for its own mode (SMEP/SMAP, supervisor shadow stacks, kernel PAC keys and
  BTI — the kernel hardens itself first) and for userspace (per-thread user
  shadow stacks, per-process PAC keys, MTE heap tagging).
- *Emitted by the toolchain everywhere.* Landing pads and PAC signing are
  code generation, so the entire tree — kernel, `libs/`, servers, drivers —
  builds with branch protection on. The instructions encode as NOPs on
  hardware without the features, so one binary runs everywhere and the
  protections light up where the silicon supports them.
- *Designed into the ABI from day one.* Thread creation allocates the shadow
  stack; signal delivery, unwinding, and `longjmp` are specified against it;
  binaries record which protections they were built with. This is where
  "design inputs, not retrofits" is priced: Linux took over a decade from CET
  silicon to shipped user shadow stacks because its ABI (`longjmp`, green
  threads, JITs) predates the mechanism — an ABI born with it pays nothing.
- *The payoff concentrates in code Rust can't check.* Safe Rust already
  statically prevents most of what these catch, so the value lands on the
  kernel's `unsafe` core, `unsafe` blocks in drivers and `libs/`, and above
  all ported C/C++ — the
  [POSIX ports](#posix-strategy-first-class-ports-without-unnecessary-change)
  and [GPL driver islands](licensing.md#why-gpl-islands-are-legally-sound)
  that are the system's least-trusted code get the full set with zero source
  changes, because everything here is codegen plus kernel enforcement. MTE
  runs synchronous in the e2e/debug harness, asynchronous in production.

**The honest limit.** These are hardening, not boundaries: they break exploit
*techniques*, and detection is probabilistic where tags and signatures are
small (MTE's 4 bits; PAC signatures shrink as virtual-address bits grow).
Hardware coverage is also uneven — CET needs ~2020-or-later x86 cores, PAC/BTI
need ARMv8.3+/8.5+, MTE is rarer still. So the baseline posture is: every
binary carries the protections, the kernel enforces whatever the silicon
offers, and a missing feature degrades defense-in-depth only — actual security
boundaries are always the MMU/IOMMU/EPT rungs of the
[isolation ladder](#intra-address-space-compartments-mpkpks-and-the-isolation-ladder).

## Boot path

```
UEFI (Secure Boot) → Limine → kernel → root server → devmgr → system servers → login
```

Every stage measures the next into the TPM; the verified chain and rollback-protected
A/B system images are specified in a future boot & update document.

## Live update model

- **Components** (drivers, servers): quiesce → serialize state → start new version →
  restore → resume, with the old binary and state blob retained for rollback. This is
  a [wire-protocol feature](driver-wire-protocol.md#62-quiesce--serialize--restore-live-update),
  not per-component heroics.
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

## Agents: the container substrate plus a grant model

AI agents acting on the user's behalf are a first-class workload
([goals](../vision/goals.md#agent-operation-natively)): a user runs an agent
holding a fraction of their authority, and that agent spawns sub-agents holding
a fraction of *its* authority, to any depth. Dovenix needs no agent-sandbox
subsystem for this, for the same reason it
[needs no container subsystem](#containers-namespaces-were-never-in-the-kernel-to-begin-with):
delegation-by-subset is the native spawn path.

- **Nesting is enforced by construction, not policy.** Handles are unforgeable
  and there is no ambient authority, so a spawned process's maximum reach is
  exactly the handle set and namespace map its spawner chose to pass — at
  every depth, with no per-level sandbox policy to get wrong. An agent is a
  process tree with a narrower namespace map and a `Job` budget (a runaway
  agent tree is throttled, accounted, and killed as a unit); a sub-agent is
  the same thing one level down.
- **Attenuation, not just subsetting.** Useful delegation passes *weakened*
  capabilities, not merely fewer of them: a read-only view of a directory, a
  network cap valid only for named hosts, a rate-limited channel. Two
  mechanisms cover it. Kernel **handle rights** express what the kernel can
  see (duplicate-with-fewer-rights: read-only, no-transfer; specified in the
  [capability model](capabilities.md#rights-kernel-legible-attenuation)).
  **Interposition proxies**
  express semantic attenuation the kernel cannot (filesystem subtree views,
  host filtering, rate limits) — the same machinery as
  [L2 record/replay](determinism.md#l2--component-recordreplay-interposition),
  with the
  [isolation ladder](#intra-address-space-compartments-mpkpks-and-the-isolation-ladder)
  pricing where the proxy runs.
- **Authority is what a process can reach, not what it holds.** A single
  channel to a powerful server is an amplifier: if a server hands out
  capabilities by global name on request, the requester's real authority is
  whatever that server will give it, and the spawn-time subset is fiction.
  Hence the load-bearing rule: **system servers never act as
  ambient-authority oracles** — anything an endpoint hands out is derived
  from and scoped by the requesting connection's namespace map, so reachable
  authority stays a subset of the spawner's reachable authority at every
  delegation step.
- **Escalation is a mediated grant, not a sandbox escape (powerbox).** Agents
  legitimately need more authority mid-task. The capability-native answer is
  a trusted, user-owned policy component: the agent asks, the user (or the
  user's standing policy) confirms, and a fresh narrow capability is minted
  and passed down the agent's existing channel. Entirely userspace, and it
  composes with nesting — a sub-agent's request is bounded by each ancestor's
  grant policy on the way up.
- **Every agent run is recordable and replayable.** L2 interposition already
  captures everything a component saw and sent; applied to an agent boundary,
  that is a complete, exactly-replayable audit of what the agent did.
  Forensics and "watch the agent" tooling fall out of the determinism
  machinery instead of a bolted-on audit subsystem.
- **Revocation is a first-class object, not a tree walk.** Handles delegated
  across a trust boundary are tethered to a `Grant`; revoking it kills that
  grant and everything delegated beneath it, transitively, while the
  delegatee keeps running. Full design — and the case against an seL4-style
  derivation tree — in the
  [capability model](capabilities.md#grants-revocation-as-an-object-not-a-tree-walk).

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

## POSIX strategy: first-class, ports without unnecessary change

Easy porting of existing Linux/BSD/macOS software — the target is
`./configure && make` with zero or trivial patches — is a stated objective. This is
where Dovenix **deliberately departs from Fuchsia**: Zircon treats POSIX as an
emulation detail ("POSIX-lite": no `fork`, no signals, no real ptys), and the
consequence is that bringing software to Fuchsia is a rewrite. Dovenix keeps the
capability-native core but inverts the priority at the boundary:

- **Capability-native underneath.** Servers speak Dovenix protocols; naming is
  per-process namespace maps; nothing in the core *requires* POSIX.
- **POSIX-complete at the libc boundary.** mlibc (born from Managarm, built for
  exactly this kind of OS) is the primary interface for ported software; `posixd`
  and `vfsd` are designed around POSIX semantics from day one, not adapted to them
  later.
- **POSIX is allowed to shape kernel primitives.** Where the Zircon model conflicts
  with efficient, correct POSIX, POSIX wins:

| POSIX requirement | Design concession |
|---|---|
| `fork()` | COW address-space snapshot as a kernel operation. Zircon refuses this; we don't. `posix_spawn` is the preferred fast path, but `fork` must be *correct*, because ported software uses it. |
| Signals | An asynchronous thread-interruption/redirection primitive (not just Zircon-style suspend + exception ports). |
| File-backed `mmap` | VMO-backed file mappings served by `vfsd`, coherent with `read`/`write`. |
| `select`/`poll`/`epoll`/`kqueue` | Readiness signaling is native in every server protocol, so polling APIs are thin, not emulated with helper threads. |
| Filesystem semantics | `vfsd` implements POSIX natively: permissions, hardlinks, atomic `rename`, unlink-while-open. Not emulation over an alien model. |
| Ptys, UNIX sockets, sessions/process groups/job control | First-class in `posixd`, because shells, terminals, and build systems are the first things ported. |

- **The namespace map is what provides the Unix world.** `/`, `/dev`, a minimal
  `/proc`, mounts — assembled per-process from the map. Ported software sees a
  normal Unix; the same mechanism is the container substrate. (Plan 9 proved this
  duality; Fuchsia uses it too — the difference is we commit to a *complete* Unix
  view.)
- **Precedent that this works**: Managarm — userspace-server OS with working
  `fork`, signals, `epoll`, ptys — runs Wayland, X, and real ported packages via
  mlibc. The pattern is proven; Dovenix's twist is the hardware-enforcement and
  live-update machinery underneath it.

**Porting metric (tracked from M3)**: a named package set (sqlite, openssh, git,
tmux, nginx, curl, …) must build and pass its own test suite unpatched or with
trivial patches. This metric is a release gate, same as the performance benchmark.

## Open questions

- Exact kernel syscall surface (needs its own doc) — including the exact
  shape of the POSIX-driven primitives: COW address-space snapshot for
  `fork`, thread interruption for signals. The rights and revocation
  semantics it must implement are fixed by the
  [capability model](capabilities.md).
- MPK/PKS compartment policy: which regions get the 16-key budget, and is
  rung 2 ever promoted to a real security boundary (gadget scrubbing + CFI)?
  Own design doc due with the first servers (M2/M3) — see the
  [compartments section](#intra-address-space-compartments-mpkpks-and-the-isolation-ladder).
- Scheduler design for the Linux-benchmark goal (needs its own doc): fair
  class + RT/deadline classes (seL4-MCS-informed budgets), IPC priority
  inheritance mechanics, core shielding, NUMA topology awareness.
- Filesystem choice for M3/M4 (port vs. new; candidates: FAT for boot, then a
  permissively-licensed modern FS or a new one).
- Namespace map format and spawn-time handoff ABI (the container substrate — needs
  its own doc before posixd lands).
- Suspend vs. live-update quiesce: same FSM states or a parallel power FSM in DWP?
  (Tracked as DWP open question; current lean: same states, different resume verb.)
- How much of the OCI runtime spec to implement natively vs. adapt via a ported
  runtime shim.

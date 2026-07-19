---
status: active
updated: 2026-07-18
depends-on: []
approved-by: pending     # awaiting human approval per plans/README rule 3
---

# M1 — Boots in QEMU

## Goal

From the roadmap: UEFI boot via Limine → kernel → root server on x86_64.
Kernel: address spaces, threads, capability handles, channels, shared-memory
objects (VMOs), interrupts. Serial console. First e2e test: boot QEMU, run
harness, power off, assert transcript.

Demo: `cargo xtask e2e m1` boots a UEFI image in QEMU, the root server (ring 3)
round-trips a message to the kernel log over a channel, and the harness asserts
the full transcript.

## Non-goals (for this plan)

- devmgr, DWP, any driver beyond the in-kernel serial console (→ M2; the
  serial `console`-class *driver host* is M2 — M1's serial is the kernel's own
  early console)
- SMP (bring-up is single-CPU; SMP is early M2 so it lands before drivers)
- `Grant` objects and revocation (tethering, revoke, grace periods —
  [capability model](../docs/src/design/capabilities.md)); M1 only shapes the
  handle-table entry so Grants land later without a relayout (see Phase 4)
- `Job` semantics (budgets, limits, kill-by-subtree) — M1 has a root Job that
  is only a process container, so the spawn path is Job-rooted from day one
- Secure/measured boot chain (design doc first; hardware milestone M4)
- ARM64 (post-M5)

## Order of work

### Phase 1 — Scaffolding and harness-first

- [ ] Workspace members: `kernel/` (no_std, x86_64-unknown-none target),
      `tools/xtask` (build/run/test driver). Outcome: `cargo xtask build`
      produces a kernel ELF.
- [ ] `tools/xtask image`: assemble ESP layout (Limine binary + config +
      kernel) into a GPT disk image. Outcome: image file builds reproducibly.
- [ ] `tools/xtask run` / `e2e`: boot image in QEMU (OVMF, `-serial stdio`,
      `isa-debug-exit`), capture serial, timeout, exit-code plumbing.
      Outcome: harness runs against a stub kernel that prints one banner line
      and exits QEMU; transcript assertion framework in `tests/system/m1/`.
- [ ] CI job: build + m1 e2e on every push. Outcome: red/green on GitHub.

### Phase 2 — Kernel bring-up

- [ ] Limine boot protocol entry: memory map, HHDM offset, framebuffer info
      (stored, unused), kernel cmdline. Outcome: banner prints memory map
      summary; harness asserts region count sanity.
- [ ] Early serial console (16550 UART) + `log`-style kernel logging with
      levels. Outcome: structured boot log; all later assertions build on it.
- [ ] GDT/IDT, exception handlers with register-dump panic path. Outcome:
      deliberate `int3` under a test cmdline flag produces asserted panic log.
- [ ] Physical frame allocator over the Limine map. Outcome: boot-time
      allocation exercise (test flag) logs asserted counters.
- [ ] Kernel address space + paging management (4-level, NX, W^X from the
      start), kernel heap. Enable the
      [baseline CPU mitigations](../docs/src/design/architecture.md#cpu-level-mitigations-the-baseline-hardening-set)
      the silicon offers, at minimum SMEP/SMAP/UMIP (a missing feature
      degrades gracefully, per the doc). Outcome: heap exercise under test
      flag; W^X self-check and enabled-mitigation list logged and asserted.

### Phase 3 — Interrupts, time, threads

- [ ] APIC (xAPIC/x2APIC) + timer, PIC masked off. Outcome: timer tick counter
      in log, asserted.
- [ ] Kernel threads + context switch + round-robin scheduler + sleep/wake.
      Outcome: two threads ping-pong a counter; transcript asserts interleaving.

### Phase 4 — Kernel object model (the Zircon-inspired core)

- [ ] Handle table + rights + generic object infrastructure, implementing the
      [capability model](../docs/src/design/capabilities.md): one-cache-line
      entries carrying the rights bitmask and the tether word (null in M1 —
      Grants are M2+, but the slot and its null check exist from day one),
      monotone attenuation on duplicate, closed acquisition (no syscall takes
      a name; koids never convert to handles). Outcome: kernel self-exercise
      under test flag (create / dup-with-subset / dup-amplification-denied /
      dup-without-DUPLICATE-denied / close / rights-deny) logged + asserted.
      (Syscall surface doc drafted alongside — see Open items.)
- [ ] VMOs + user address spaces (map/unmap/protect). Outcome: exercised via
      first user process in Phase 5 (no earlier observable).
- [ ] Channels (message + handle transfer) + Events. Events are latching
      edges from day one (signal marks pending, wait consumes, edges
      coalesce) and a wait on a dead peer returns the terminal error rather
      than hanging — these are the semantics the
      [DWP doorbell contract](../docs/src/design/driver-wire-protocol.md#84-doorbells)
      assumes in M2, and retrofitting latching is a redesign. Outcome:
      kernel-side loopback exercise asserted (including a coalesced-signal
      and a wait-after-peer-close case); real proof is Phase 5.

### Phase 5 — Userspace

- [ ] Syscall entry/exit (syscall/sysret), per-thread kernel stacks, usercopy
      discipline (SMAP-guarded, MTE-shaped for later ARM; user memory is
      [read once into a snapshot before validation](../docs/src/design/principles.md#security-and-isolation),
      never re-read after checking). Outcome: a
      hand-built "syscall abuse" user blob under test flag exercises
      bad-pointer/bad-handle paths; asserted error codes, no panics.
- [ ] Process + thread spawn from an initrd module (flat ELF loader; initrd =
      extra Limine module). Spawn hangs off the root Job and passes an
      explicit startup handle set — acquisition path 4 of the
      [closed acquisition rules](../docs/src/design/capabilities.md#no-global-namespaces-handle-acquisition-is-closed).
      Outcome: `hello` user process prints via debug-log syscall; transcript
      asserted.
- [ ] Root server: spawned with the initial handle set from the kernel — the
      base case of the acquisition graph; round-trips a message kernel↔user
      over a channel; clean system power-off syscall (gated to the root
      server's handle). Outcome: **the M1 demo transcript** — this is the
      milestone gate.

## Verification

- `tests/system/m1/`: one QEMU boot per scenario group; assertions are
  transcript-based (golden-ish but tolerant of noise: ordered required lines).
  Scenarios: clean boot, memory exercises, exception path, scheduler
  interleave, handle exercises, syscall abuse, full root-server demo.
- Every task above lands with its assertion in the same PR (plans/README rule 5).
- Gate: all m1 scenarios green in CI on a clean checkout.

## Deviations

<append-only>

## Open items

- Design constraints on Phase 3/4 from the determinism & multicore docs
  (added 2026-07-17): scheduler and channel design must not preclude IPC
  priority inheritance (thread/channel objects carry an effective-priority
  slot from the start); kernel state per-CPU-shaped even while single-CPU;
  all time exposed to userspace flows through kernel-mediated interfaces.

- Design constraints on Phase 4/5 from the
  [capability model](../docs/src/design/capabilities.md) (added 2026-07-18):
  the error space reserves `ERR_PEER_RESET` and `ERR_REVOKED` as distinct
  codes with distinct retry classes from the first syscall, even though
  nothing revokes in M1; the provisional rights table (universal +
  VMO/Channel/Process/Thread rights, plus Job `SPAWN` for the root Job) is
  the constant set M1 implements — the remaining Job rights and
  Interrupt/IoRegion/VmDomain rights are reserved values, not code.
  Phase 2's paging design keeps PTE protection-key bits available for the
  [isolation ladder](../docs/src/design/architecture.md#intra-address-space-compartments-mpkpks-and-the-isolation-ladder)
  (MPK is M3+; burning the bits now would foreclose it). Phase 5's thread
  ABI is shadow-stack-shaped: thread creation has a slot for shadow-stack
  allocation even though CET itself waits for a later milestone —
  ["designed into the ABI from day one"](../docs/src/design/architecture.md#cpu-level-mitigations-the-baseline-hardening-set)
  is exactly the retrofit cost the book warns about.

- Syscall ABI doc (`docs/src/design/syscalls.md`) should be drafted during
  Phase 4 while the handle model is hot — it's a book doc, so design review
  happens there, not here. The
  [capability model](../docs/src/design/capabilities.md) declares itself
  normative for this doc: names and constants land there, semantics are
  already fixed.
- Limine version pinning + vendoring policy (binary in-tree vs. fetched by
  xtask; lean: fetched + checksummed, keeping the tree source-only).
- QEMU/OVMF versions for CI runners — pin in xtask, document in tools/README.

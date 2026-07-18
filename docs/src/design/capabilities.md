# Capability Model: Handles, Rights, Revocation

This chapter resolves two questions the
[agent model](architecture.md#agents-the-container-substrate-plus-a-grant-model)
left open — what a *weakened* handle is (the rights model) and how a grant is
*taken back* (revocation) — and states the acquisition rules behind the
kernel-philosophy line "no global namespaces in the kernel". It is normative for the future syscall-surface
document — the exact syscall names and constants land there, but they must
implement the semantics here.

## The handle model

A handle is an entry in a per-process kernel handle table:

```
handle ──▶ { object ref, rights bitmask, tether ──▶ Grant | none }
```

Processes name entries by process-local integers; the kernel owns the table,
so handles are unforgeable and everything below is enforced at the syscall
boundary. The table entry is one cache line; rights and tether checks read
the same line the handle lookup already fetched, so their marginal cost on
the hot path is a predictable branch, not a memory access.

## No global namespaces: handle acquisition is closed

"No global namespaces in the kernel; namespaces are a userspace concern"
([kernel philosophy](architecture.md#kernel-philosophy-zircon-inspired-hardware-enforced))
is not a style preference — it is two enforceable rules of the handle model.

**Rule 1 — the kernel resolves no names.** No syscall takes a name — a path,
a string, a registered port, a well-known identifier of any kind — and
returns a handle. There is no kernel registry to publish into or look up
from. A process acquires a handle in exactly four ways, and no fifth may
ever be added:

1. **Create** a new kernel object (full rights for the type, to the creator);
2. **Duplicate** a handle it already holds (rights a subset of the source);
3. **Receive** a handle in a channel message;
4. **Spawn**: receive the startup handle set its spawner assembled.

Every path except creation starts from a handle some process already held,
and the base case is the root server, which receives the initial handle set
from the kernel at [boot](architecture.md#boot-path). So the acquisition
graph is rooted in explicit grants: tracing any handle backwards ends in
the spawn chain, never in a lookup. This closure is what makes
[monotone attenuation](#rights-kernel-legible-attenuation) mean something —
a kernel name service would be the backdoor around it, because a process's
maximum reach would become "everything nameable" rather than "everything
granted".

**Rule 2 — identity is not authority.** Kernel objects may carry global
identifiers (koids) for accounting, tracing, and debugging. These are pure
data: no operation converts an identifier back into a handle, so learning
an object's koid grants nothing. Introspection and debugging tools acquire
their handles the same four ways as everyone else.

"Namespaces are a userspace concern" is the constructive half: every name
in the system — file paths, service names, device paths, network
identities, POSIX-visible PIDs — is a userspace server's *interpretation*
of the requesting connection's
[namespace map](architecture.md#containers-namespaces-were-never-in-the-kernel-to-begin-with).
To the kernel, a namespace map is opaque data passed at spawn (its format
and handoff ABI are a
[tracked open question](architecture.md#open-questions)). Name resolution
is therefore always *scoped*: a server answering a lookup derives the
result from that connection's map — never from a global table, per the
[no-oracle rule](architecture.md#agents-the-container-substrate-plus-a-grant-model).
The consequences the rest of the book stands on — containers with no
"namespace escape" attack class, agent delegation bounded at every depth,
the per-process Unix world view — all reduce to these two rules.

## Rights: kernel-legible attenuation

Rights are a bitmask carried **by the handle, not the object** — two handles
to one VMO can differ in what they permit. Every syscall declares the rights
it requires; the check happens at handle-table lookup.

The rules, all of which exist to make delegation safe to reason about:

- **Creation grants full rights** for the object's type, to the creator only.
- **Attenuation is monotone.** `duplicate` accepts a rights mask that must be
  a subset of the source handle's; `transfer` moves a handle with rights
  intact. There is no amplification operation, and none may ever be added —
  a process's rights over an object can only shrink along any chain of
  duplicates and transfers. Combined with unforgeability this yields the
  [agent-model invariant](architecture.md#agents-the-container-substrate-plus-a-grant-model)
  at the kernel layer: a delegatee's authority is bounded by its delegator's,
  at every depth, by construction.
- **`DUPLICATE` and `TRANSFER` are themselves rights.** A handle without
  `DUPLICATE` cannot fan out; one without `TRANSFER` is confined to the
  process holding it. (Honesty note: `!TRANSFER` confines the *handle*, not
  the data reachable through it — information exfiltration is a covert-channel
  problem, out of scope for the rights model.)

### What rights deliberately don't do

Rights are the **kernel-legible** subset of attenuation: coarse, per-type,
fixed at design time. Semantic scoping — a filesystem subtree, a set of
permitted hosts, a rate limit — is invisible to the kernel and must stay
that way; encoding semantics into kernel rights would grow the syscall
surface without bound. Semantic attenuation belongs to
[interposition proxies](architecture.md#agents-the-container-substrate-plus-a-grant-model),
priced by the
[isolation ladder](architecture.md#intra-address-space-compartments-mpkpks-and-the-isolation-ladder).
The kernel's job ends at making the coarse rights cheap and the proxies
possible.

### Provisional rights table

Final constants land with the syscall-surface document; additions are
minor-version material, removals are not.

| Scope | Rights |
|---|---|
| Universal | `DUPLICATE`, `TRANSFER`, `WAIT`, `SIGNAL`, `INSPECT` |
| VMO | `READ`, `WRITE`, `EXECUTE`, `MAP`, `RESIZE` |
| Channel | `SEND`, `RECEIVE` |
| Process / Thread | `MANAGE`, `READ_STATE`, `WRITE_STATE` |
| Job | `SPAWN`, `KILL`, `BUDGET` |
| Interrupt | `WAIT`, `ACK` |
| IoRegion | `MAP` |
| VmDomain | `CONFIGURE`, `ENTER` |

## Revocation

The requirement comes from the
[agent model](architecture.md#agents-the-container-substrate-plus-a-grant-model):
a user must be able to yank a running delegatee's authority mid-flight —
one grant or a whole delegation subtree — without necessarily killing it,
promptly and transitively.

### Why not a derivation tree

The classic answer is seL4's capability derivation tree: the kernel records
every derivation as a tree edge and revocation walks the descendants. It is
precise, and it is disqualified here twice over:

- **It is a globally-written structure on the hot path.** Handles are
  duplicated and transferred as part of ordinary IPC — every channel message
  carrying a handle would insert into a shared tree. seL4 affords this under
  its big kernel lock; [multicore-first](principles.md#concurrency-and-determinism)
  forbids exactly this shape: kernel state per-CPU, never behind new global
  locks, and no contended write on a path goal 3 benchmarks against Linux.
- **It taxes every handle to serve a rare event.** Per-handle derivation
  metadata is paid always; revocation is exceptional. The cost belongs on
  the delegation boundary, where the [agent model](architecture.md#agents-the-container-substrate-plus-a-grant-model)
  already does its bookkeeping.

### Grants: revocation as an object, not a tree walk

A **Grant** is a kernel object representing one revocable delegation:

- **Tethering.** At delegation time, the delegator attaches handles to a
  Grant it holds; a tether is sticky — every duplicate or transfer of a
  tethered handle carries the tether. There is no untether operation; a
  tethered handle may only be re-tethered to a *child* of its current Grant.
- **Nesting.** Tethering an already-tethered handle to a new Grant links the
  new Grant under the old one. Delegation chains therefore build a small
  grant graph — one node per trust boundary, not per handle — and revoking a
  Grant revokes its descendants: authority a sub-agent received *through* a
  revoked delegation dies with it, transitively, with no per-handle tracking.
- **Check-on-use.** The handle-table entry's tether pointer is followed at
  lookup: one acquire-load of a read-mostly liveness word plus a branch.
  Untethered handles (the overwhelming majority — system servers talking to
  each other) pay a null check on a line already in cache. This is the
  zero-cost-when-off discipline applied to revocation.
- **Revoke.** Marks the Grant and its descendants dead (a walk over dozens
  of grants, not millions of handles), then waits an RCU-style grace period —
  every CPU passes a schedule point — before returning, so no thread is still
  inside a syscall admitted under the dead grant. No lock is shared with the
  use path; readers never block the revoker, the revoker never blocks readers.
- **Memory is the hard case, and it is handled eagerly.** A mapping created
  through a tethered `MAP`-right VMO handle is registered on the Grant;
  revoke unmaps it (TLB shootdown — expensive, rare, inherent) so revoked
  memory is gone, not merely unavailable to new syscalls. Shared-memory rings
  die the same way.

### What revocation looks like to the revoked

From the revoked side, a dead grant is a peer that went away: syscalls on
tethered handles return `ERR_REVOKED`, ring doorbells stop, mappings fault.
This is deliberately the shape clients already survive — the
[peer-reset recovery discipline](principles.md#protocols-and-lifecycle) —
with one difference: `ERR_REVOKED` is distinct from `ERR_PEER_RESET` and its
retry class is always "don't retry; re-acquire authority (powerbox) or fail
upward." Revocation timing is asynchronous to the victim, but it satisfies
the [determinism invariants](determinism.md): the revoke itself is a recorded
syscall, and every point where the victim observes it (error return, fault)
is a kernel-mediated, recordable input — revoked runs replay exactly.

### Choosing the rung

Revocation has a cost ladder, like isolation; pick the cheapest rung that
meets the need:

1. **Close the proxy** — semantic grants mediated by an interposition proxy
   die when the proxy drops the session. No kernel involvement.
2. **Revoke the Grant** — kernel-enforced, transitive, leaves the delegatee
   running (it can ask the powerbox for re-grant or wind down gracefully).
3. **Kill the `Job` subtree** — the coarse fallback; revokes everything by
   ending the holder.

## Open questions

- Exact rights constants and per-syscall required-rights tables (syscall
  document).
- Grant granularity conventions: one session grant per agent plus one per
  powerbox escalation is the working model; whether finer defaults are worth
  the bookkeeping is a userspace policy question, not a kernel one.
- Whether `ERR_REVOKED` should carry the dead Grant's identity for
  diagnostics, and how that interacts with not leaking delegation topology
  to the revoked.

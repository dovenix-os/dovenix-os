# Driver Wire Protocol (DWP)

> **Status: draft v0.** This is the load-bearing interface of Dovenix. It is
> deliberately specified before any code exists.

## 1. Why this protocol carries the whole system

Every driver in Dovenix — native, ported-GPL, or backed by the Linux driver VM — is
an isolated process whose **only** connection to the rest of the system is this
protocol. That single boundary serves [four goals](../vision/goals.md) at once:

1. **License boundary** ([licensing model](licensing.md)): a GPL-2.0-only ported driver is a separate
   program communicating over a versioned wire protocol. The GPL stops here.
2. **Fault isolation** (goal 2): a crashed driver is restarted by its supervisor and
   re-driven through the same protocol; clients observe a defined error, not a hang.
3. **Live update with rollback** (goal 4): upgrade is quiesce → serialize → restart
   new binary → restore, expressed as protocol states. Crash-restart is the same path
   with an empty state blob.
4. **Testability** (goal 5): a driver is tested by speaking DWP to it from a harness;
   a client is tested against a fake driver. Golden transcripts and fuzzing target
   the wire format directly.

Consequently: **changes to this protocol are the most expensive changes in the
project.** Compatibility rules (§7) are non-negotiable.

## 2. Roles and topology

- **devmgr** (supervisor): spawns driver hosts, grants device resources, owns the
  driver lifecycle. Every driver host has exactly one *supervisor channel*.
- **Driver host**: one process per driver instance.
- **Clients**: system servers (vfsd, netd, …) or other drivers (a bus driver is the
  supervisor-delegate for its children). Each client ↔ driver connection is a
  dedicated *service channel* pair, obtained from devmgr by capability handoff —
  drivers never accept unsolicited connections.

Each channel carries exactly one DWP session. A driver may serve many service
channels concurrently but has exactly one supervisor channel.

## 3. Transport assumptions

DWP is defined over the
[kernel's primitives](architecture.md#kernel-objects-initial-set):

- **Channel**: datagram-oriented, bidirectional, preserves message boundaries,
  transfers capability handles along with bytes. Control plane.
- **VMO + Event**: shared-memory regions with doorbell signaling. Data plane
  (§8) — bulk payloads never travel through channel messages.

The protocol is asynchronous throughout: requests carry transaction IDs and
responses may arrive out of order (§5).

## 4. Wire format

All integers are little-endian. Every channel message begins with a fixed 32-byte
header:

```text
offset  size  field
0       4     magic        = 0x44575030  ("DWP0")
4       2     proto_major
6       2     proto_minor
8       4     msg_type     (class-scoped, §9; top bit set = response)
12      8     txid         (request/response correlation; 0 = event/no reply)
20      4     payload_len  (bytes following the header)
24      4     status       (0 in requests; DwpStatus in responses)
28      2     n_handles    (capability handles attached to this message)
30      2     flags
```

The payload is a flat, versioned struct per `msg_type`, defined in the protocol IDL
(§10). Unknown trailing payload bytes MUST be ignored (this is how minor versions
add fields); unknown `msg_type` MUST produce `ERR_NOT_SUPPORTED`, never a session
abort.

## 5. Transactions

- Requests carry a non-zero `txid` unique per session among in-flight requests.
- Responses echo the `txid` and set the response bit in `msg_type`.
- Messages with `txid = 0` are one-way events (either direction).
- Ordering: responses are unordered relative to each other. Where a driver class
  needs ordering (e.g., barriers in block), it is expressed explicitly in that
  class's messages, never assumed from the transport.
- Every request has a bounded outcome: response, `ERR_CANCELED` (client closed),
  or `ERR_PEER_RESET` (driver crashed/restarted, §6.3). Nothing hangs forever.

## 6. Lifecycle state machine

The supervisor channel drives every driver host through this FSM:

```text
        spawn                 BIND_OK               START
  ┌──────────┐  HELLO ok  ┌──────────┐  resources ┌─────────┐
  │  INIT    │──────────▶│  BOUND    │──────────▶│ RUNNING  │◀────────────┐
  └──────────┘            └──────────┘            └────┬─────┘             │
                                                       │ QUIESCE           │ RESTORE ok
                                                       ▼                   │
                                                  ┌──────────┐   SAVE  ┌───┴──────┐
                                                  │QUIESCING │───────▶│ QUIESCED  │
                                                  └──────────┘         └───┬──────┘
                                                    (drains in-flight)     │ STOP (new binary spawned,
                                                                           ▼  state blob handed over)
                                                                      [exit / handoff]
```

### 6.1 Bring-up

1. `HELLO` (§7): version negotiation, driver identity, capabilities advertisement.
2. `BIND`: devmgr passes device resources as handles — IoRegions, Interrupts,
   [IOMMU-constrained DMA authority](architecture.md#dma-isolation-and-the-registration-model),
   config data. The driver never enumerates or
   claims hardware itself; **all device authority arrives explicitly on this
   channel.** (Security goal: a driver's maximum reach is what BIND granted.)
3. `START`: driver begins serving service channels.

### 6.2 Quiesce / serialize / restore (live update)

- `QUIESCE(deadline)`: driver stops accepting new work, drains or parks in-flight
  operations, brings the device to a suspendable state. Overrunning the deadline is
  a protocol violation → supervisor escalates to kill + crash-restart path.
- `STATE_SAVE`: driver returns an opaque, **versioned** state blob (§6.4) via VMO.
- Supervisor spawns the new binary, replays `HELLO`/`BIND`, then issues
  `STATE_RESTORE(blob)` instead of a cold `START`.
- **Rollback**: the old binary and blob are retained until the new instance reports
  healthy (`START`/`RESTORE` acknowledged and a supervisor-defined probation
  passes). Restore failure → automatic rollback to the old binary + blob.

### 6.3 Crash-restart

Crash-restart is the degenerate live update: no blob, `BIND` replayed from devmgr's
retained configuration, cold `START`. On every service channel the driver held,
clients receive `ERR_PEER_RESET` for in-flight transactions and an event announcing
the restart epoch. **Client contract**: every driver class defines, per operation,
whether it is safe to retry after `ERR_PEER_RESET` (idempotent) or must be
re-validated (e.g., a block write that may or may not have hit media). This is
specified per-message in the class IDL, not left to client guesswork.

### 6.4 State blobs

- Opaque to the supervisor; versioned by the driver:
  `(driver_id, state_major, state_minor, payload)`.
- A new driver binary MUST restore blobs with the same `state_major` and any lower
  or equal `state_minor`, MAY refuse otherwise (→ rollback).
- Blobs contain only serializable logical state (queues, negotiated features,
  client sessions) — never raw pointers, handles, or device register snapshots that
  can be re-derived.

## 7. Versioning and negotiation

- The **core protocol** (this document: header, lifecycle, transactions) and each
  **driver class** (§9) carry independent `major.minor` versions.
- `HELLO` advertises, per side, the supported range; the negotiated version is the
  highest mutually supported. No overlap → `ERR_INCOMPATIBLE`, and devmgr treats
  the driver as unusable (never "best effort").
- **Minor** versions only append: new messages, new trailing payload fields, new
  enum values. Receivers ignore what they don't know.
- **Major** bumps are breaking and expected to be *rare and dual-served*: during a
  transition, drivers serve N and N-1. This is what makes live-updating a client
  and its driver independently possible.

## 8. Data plane: generic queues

Bulk data (block I/O, packets, frames, input events at rate) moves over shared
memory, virtio-inspired. The queue machinery — memory layout, index discipline,
doorbells, validation — is defined **once, here**, and implemented once
(`libs/dwp`, shared and fuzz-hardened). A driver class contributes only its
descriptor/completion entry formats and the semantics of its operations.

### 8.1 Queue anatomy

A *queue* = one ring VMO + one doorbell Event pair + zero or more data VMOs,
created by the client and granted to the driver in a class-specific
`*_QUEUE_CREATE` message, with handles in this canonical order:

```text
[0]   ring VMO
[1]   submit doorbell Event    (client → driver)
[2]   complete doorbell Event  (driver → client)
[3..] data VMOs                (bulk buffers; DMA-granted via IOMMU)
```

Every queue is a **submission ring + completion ring** pair. What the entries
*mean* is a class concern (a block submission is an I/O request; a net RX
submission is an empty buffer being posted) — the machinery is identical.

### 8.2 Ring VMO layout

Page-aligned:

```text
header page:
  0     u32  submit_producer     (client-written)
  64    u32  submit_consumer     (driver-written)
  128   u32  complete_producer   (driver-written)
  192   u32  complete_consumer   (client-written)
submission ring:  depth × S-byte class descriptor
completion ring:  depth × C-byte class completion entry
```

- `depth` is a power of two, negotiated at queue creation.
- Each index owns a cache line; single producer / single consumer per index.
- Indices are free-running (monotonic), masked by `depth - 1` to a slot —
  full vs. empty is unambiguous without a wasted slot.
- `S` and `C` are fixed constants of the class (per major class version).

### 8.3 Descriptor conventions

- Every submission's first field is a client-chosen `req_id: u64`, unique among
  in-flight entries on that queue; every completion echoes it.
- Completions may be delivered out of submission order. Any stronger ordering
  guarantee is stated explicitly by the class, never assumed.
- Unknown flag bits MUST be zero on submission and MUST produce an error
  completion — never a crash, never silent acceptance.

### 8.4 Doorbells

Edge signals, both directions. Both sides MUST tolerate spurious wakeups and
coalesced edges: **index comparison is ground truth**. This is what makes
polling mode and interrupt mitigation configuration-free client policies.

### 8.5 Trust and validation

Each side treats ring and descriptor contents as **adversarial** — rings cross
a fault boundary and (for GPL islands and VM-backed drivers) a trust boundary.
The consumer re-validates every field at consumption time (opcodes, ranges,
alignment, data-VMO bounds). A malformed descriptor produces an error
completion; a corrupt index (out of window, regressing) costs the offender the
session. Neither side may be crashable, blockable, or hangable by the other's
ring behavior.

### 8.6 Versioning

This section's layout is versioned with the **core protocol**. Descriptor and
completion formats are versioned with their **class** (per §7: class minor
versions may only append meaning via flag/reserved fields — entry sizes are
fixed per major class version).

## 9. Driver classes

The core protocol is class-agnostic. A *driver class* = a `msg_type` namespace +
payload schemas + queue descriptor/completion formats (the machinery itself is
core, §8) + retry semantics, layered on the core. Initial classes:

| Class | v0 scope | First implementation |
|---|---|---|
| [`block`](classes/block.md) | read/write/flush/discard/write-zeroes, FUA + flush ordering (no barriers), geometry | virtio-blk, then NVMe |
| [`net`](classes/net.md) | tx/rx queues, MAC/link/MTU/filters, csum/TSO offload, loss-not-corruption contract | virtio-net |
| [`input`](classes/input.md) | credit-based inline event stream, evdev-inspired vocabulary, state resync | virtio-input |
| [`console`](classes/console.md) | reliable ordered byte stream, UART params, resize/break events | serial (M1), virtio-console |
| `bus` | enumerate children, grant resources (devmgr delegate) | PCIe |
| `display` | modeset + framebuffer present (deliberately minimal) | virtio-gpu |
| `battery` | charge state, health, charge-control (powerd client) | ACPI battery |
| `thermal` | zones, trip points, cooling devices (powerd client) | ACPI thermal |

Each class gets its own spec page as it is implemented. The
[`block` class spec](classes/block.md) is written and serves as the template —
including the per-operation `ERR_PEER_RESET` retry table (§6.3) and the
restart durability rule that makes crash-restart and live update safe for
storage clients.

## 10. IDL and code generation

Payload schemas are defined once in a small IDL (likely: Rust type definitions in a
`dwp-schema` crate serving as source of truth, with codegen for the wire layer) and
compiled into codec crates. Requirements:

- Deterministic flat layout (no self-describing tag soup on the hot path).
- Generated encode/decode with fuzzers derived from the same schema.
- **License**: schema and codec crates are `Apache-2.0 OR MIT` — they are linked by
  GPL islands, which is legal precisely because MIT is GPLv2-compatible. These
  crates must never depend on anything GPL.

## 11. Security posture

- No ambient authority: a driver's reach = handles received via `BIND`, period.
- DMA is IOMMU-constrained to explicitly granted regions; a driver class cannot
  request "all of memory".
- All wire input is adversarial: decoding is fuzzed, lengths are validated, and a
  malformed message from a client costs the client its channel, never the driver.
- Supervisor channel > service channel: lifecycle messages arrive only from devmgr;
  service channels cannot quiesce, rebind, or introspect a driver.

## 12. Testing hooks

- **Conformance suite per class**: a harness that speaks DWP to any driver of the
  class and asserts the FSM, version negotiation, quiesce under load, restore
  fidelity, and `ERR_PEER_RESET` semantics. Passing conformance is the definition
  of "done" for a driver.
- **Golden transcripts**: recorded channel sessions replayed in CI against new
  driver builds; wire-format regressions fail loudly.
- **Fuzzing**: schema-derived structured fuzzers for codecs; ring-state fuzzers for
  the data plane.
- **Fault injection**: the conformance harness kills drivers mid-operation and
  asserts client-visible behavior matches each operation's declared retry class.

## 13. Open questions (v0 → v1)

- Exact `DwpStatus` error code space and how driver-class errors extend it.
- Multi-queue (per-CPU) conventions for `net`/`block` and their interaction with
  quiesce.
- Whether `STATE_SAVE` supports incremental/streamed blobs for drivers with large
  state (e.g., a future GPU class).
- Power management verbs (suspend/resume overlap with quiesce — same states or
  parallel FSM?).
- Whether the Linux driver VM's virtio devices are consumed as plain virtio (kernel
  VmDomain plumbing) or wrapped by a thin DWP proxy for uniform supervision.
  Current lean: DWP proxy, so *everything* is supervisable and testable the same way.

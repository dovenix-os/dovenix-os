# Driver Class: `block`

> **Status: draft v0.1.** First DWP driver class to be specified — it is the
> **template** for all class specs: future classes follow this document's structure
> (identity → control plane → data plane → semantic contracts → lifecycle → state →
> conformance → open questions).

## 1. Scope and roles

Fixed-size-block random-access storage: virtio-blk (M2), NVMe and AHCI (M4).

- **Driver**: a driver host exposing one block device per service channel session.
  A multi-namespace/multi-LUN device exposes one session per logical device.
- **Client**: typically `vfsd` or a filesystem server; also the image builder,
  swap, and test harnesses. One client per session; fan-out (partitions, volume
  management) is a *client-side* concern layered above this class.

Out of scope for v0: zoned devices, inline encryption, request priorities,
multi-queue (one queue pair per session in v0 — see §9).

## 2. Class identity

| | |
|---|---|
| Class name | `block` |
| Class version | `0.1` (negotiated in `HELLO` per DWP §7) |
| `msg_type` namespace | `0x0001_xxxx` |

## 3. Control plane (channel messages)

All messages ride the DWP header (core §4); payloads below are the flat structs.
Sizes in blocks are in **logical blocks** unless stated.

### 3.1 `BLK_GET_INFO` (client → driver)

No payload. Response:

```rust
struct BlkInfo {
    logical_block_size: u32,   // bytes; power of two, >= 512
    physical_block_size: u32,  // bytes; >= logical_block_size
    capacity_blocks: u64,
    max_transfer_blocks: u32,  // per request
    features: u64,             // bitmap, §3.2
    discard_granularity: u32,  // blocks; 0 if discard unsupported
    discard_alignment: u32,    // blocks
    max_queue_depth: u16,
    _reserved: [u8; 6],
}
```

### 3.2 Feature bits

| Bit | Feature | Meaning |
|---|---|---|
| 0 | `READ_ONLY` | Writes rejected with `BLK_ERR_READ_ONLY` |
| 1 | `WRITE_CACHE` | Device has a volatile write cache (durability contract §5 applies in full) |
| 2 | `FUA` | Per-request force-unit-access supported |
| 3 | `DISCARD` | `BLK_OP_DISCARD` supported |
| 4 | `WRITE_ZEROES` | `BLK_OP_WRITE_ZEROES` supported |
| 5 | `ROTATIONAL` | Hint for client-side scheduling |

`BLK_SET_FEATURES(requested: u64) → enabled: u64` lets the client opt in;
the driver answers with the accepted subset. Must precede queue creation.

### 3.3 `BLK_QUEUE_CREATE` (client → driver)

Payload + handles:

```rust
struct BlkQueueCreate {
    queue_id: u16,      // v0: must be 0
    depth: u16,         // power of two, <= max_queue_depth
    data_vmo_count: u16,
    _reserved: u16,
}
// attached handles, in order:
//   [0] ring VMO        (layout: §4)
//   [1] doorbell Event  (client → driver: "requests available")
//   [2] completion Event(driver → client: "completions available")
//   [3..] data VMOs     (read/write-mapped by driver; DMA-granted via IOMMU)
```

The driver validates the ring VMO size against `depth`, maps everything, and
responds. `BLK_QUEUE_DESTROY(queue_id)` tears it down after draining.

### 3.4 Events (driver → client, `txid = 0`)

| Event | Meaning |
|---|---|
| `BLK_EV_CAPACITY_CHANGED { capacity_blocks }` | Device resized (client re-runs `GET_INFO`) |
| `BLK_EV_MEDIA_CHANGED` | Removable media swap; all cached state invalid |

## 4. Data plane (ring layout, per DWP §8)

One queue = one ring VMO + registered data VMOs + one doorbell Event pair. The
ring VMO layout, page-aligned:

```text
header page:
  0   u32  req_producer      (client-written)
  4   u32  req_consumer      (driver-written)
  8   u32  comp_producer     (driver-written)
  12  u32  comp_consumer     (client-written)
  (each index in its own cache line; monotonically increasing, masked by depth)
request ring:    depth × 64-byte BlkRequest
completion ring: depth × 16-byte BlkCompletion
```

```rust
struct BlkRequest {
    req_id: u64,        // client-chosen; echoed in completion; unique in-flight
    opcode: u16,        // BLK_OP_*
    flags: u16,         // bit 0: FUA (if negotiated)
    data_vmo: u16,      // index into registered data VMOs
    _reserved: u16,
    lba: u64,           // first logical block
    block_count: u32,
    _reserved2: u32,
    data_offset: u64,   // byte offset in data VMO; logical-block aligned
    _pad: [u8; 24],
}

struct BlkCompletion {
    req_id: u64,
    status: u32,        // BLK_OK or BLK_ERR_* 
    _reserved: u32,
}
```

Opcodes: `BLK_OP_READ`, `BLK_OP_WRITE`, `BLK_OP_FLUSH`, `BLK_OP_DISCARD`,
`BLK_OP_WRITE_ZEROES`. For `FLUSH`, `lba`/`block_count`/data fields are zero.

Rules:

- **Single contiguous data segment per request** in v0 (no SGL). Clients coalesce
  or split; vectored I/O revisited with real measurements (§9).
- Single producer / single consumer per ring direction; doorbells are edge
  signals; **index comparison is ground truth** (spurious wakeups tolerated,
  enables polling and interrupt mitigation).
- Both sides treat ring contents as **adversarial** (DWP §8): the driver
  re-validates every descriptor (opcode, ranges, alignment, VMO bounds) at
  consumption time; a malformed descriptor completes with an error, it never
  faults the driver. Out-of-range indices cost the client its session.
- Completions may be reordered relative to submissions. **There is no implicit
  inter-request ordering** — no barriers. Ordering is expressed only via
  completion-then-submit and `FLUSH`/FUA (§5). (Matches modern Linux practice;
  barrier requests were removed from the block layer for good reason.)

## 5. Durability contract

This section is normative; filesystems are built on it.

1. A completed `READ` returned data consistent with all writes completed before
   its submission.
2. A completed `WRITE` is **not durable** if `WRITE_CACHE` is present — it may
   live in a volatile cache. It is durable if: FUA was set (and negotiated), or a
   later `FLUSH` completed successfully, or the device has no `WRITE_CACHE`.
3. A completed `FLUSH` guarantees durability of **every write whose completion
   the client observed before submitting the flush**. Writes still in flight at
   flush submission are *not* covered.
4. `DISCARD` is advisory (blocks may subsequently read as anything, stable);
   `WRITE_ZEROES` is a real write and follows rule 2.

## 6. Crash-restart and retry semantics (per DWP §6.3)

On `ERR_PEER_RESET`, every in-flight request is failed and the session enters a
new restart epoch. Per-operation retry classes:

| Opcode | Retry class | Rationale |
|---|---|---|
| `READ` | **Safe to retry** | Idempotent |
| `WRITE` | **Safe to retry** | Rewriting identical data to the same LBA is idempotent |
| `WRITE_ZEROES` | **Safe to retry** | Idempotent |
| `DISCARD` | **Safe to retry** | Idempotent (advisory) |
| `FLUSH` | **Safe to retry — but see rule below** | The flush itself is idempotent; its *coverage* is not |

**The restart durability rule**: a driver restart may lose the device's volatile
cache (and with it, completed-but-unflushed writes). After observing
`ERR_PEER_RESET`, the client MUST consider **all writes not covered by an
acknowledged `FLUSH` as never having happened**, re-issue them, and re-flush.
Clients that need this to be cheap keep a bounded un-flushed write log — which
well-designed filesystems (journals, CoW) already have.

This rule is what makes crash-restart (goal 2) and driver live update (goal 4)
*safe* rather than merely available: correctness never depends on driver-side
volatile state surviving.

## 7. Quiesce, state, restore (per DWP §6.2)

- **`QUIESCE`**: stop consuming new ring descriptors; complete all in-flight
  requests; if `WRITE_CACHE`, issue a device-level cache flush; then report
  `QUIESCED`. Result: rings drained, media durable, device idle.
- **State blob** (versioned, per DWP §6.4) contains logical session state only:
  negotiated class version and features, queue configs (`queue_id`, depth,
  data-VMO count), restart-epoch counter. Ring indices live in the shared ring
  VMO and survive by construction; channel and VMO handles are re-granted by
  devmgr at restore, never serialized.
- **`STATE_RESTORE`**: new driver instance re-maps re-granted VMOs, validates
  blob against ring-VMO reality (mismatch → `ERR_STATE_INVALID` → rollback),
  re-probes hardware, resumes doorbell processing. Because quiesce drained and
  flushed, restore has zero in-flight ambiguity — the live-update path is
  *stronger* than crash-restart, and clients notice at most a latency blip.

## 8. Conformance suite (per DWP §12; definition of "done")

1. **Geometry & validation**: `GET_INFO` sanity; rejection of unaligned,
   out-of-range, oversized, and unknown-opcode requests without driver crash.
2. **Data integrity**: write/readback patterns across block sizes, offsets, and
   queue depths, including capacity boundaries.
3. **Durability**: flush semantics under simulated power-cut (QEMU
   `blkdebug`-style fault injection): nothing covered by an acked flush is lost;
   uncovered writes may be lost but never torn below logical-block granularity.
4. **Adversarial rings**: fuzzed descriptors, hostile indices, doorbell storms —
   driver must degrade to error completions or session drop, never crash or hang.
5. **Crash-restart**: harness kills the driver at randomized points under load;
   asserts every in-flight request resolves `ERR_PEER_RESET`, the retry table
   holds, and the §6 durability rule is sufficient for a journaling client model.
6. **Live update under load**: quiesce/save/restore with I/O hammering; asserts
   drain within deadline, zero lost acknowledged-and-flushed data, bounded stall.

## 9. Open questions (v0.1 → v0.2)

- **Multi-queue** (per-CPU queue pairs): layout is ready (`queue_id` exists);
  deferred until the M4 NVMe driver gives real contention numbers. Interaction
  with quiesce (drain N queues) to be specified then.
- **SGL / vectored requests**: v0's single-segment rule trades an occasional
  client-side copy for a much simpler adversarial-validation surface. Revisit
  with M3 filesystem measurements.
- **Zoned block devices** (ZNS): likely a `block.zoned` extension class, not
  feature bits on this one.
- **Removable media** flow (`BLK_EV_MEDIA_CHANGED`) needs a full spec pass with
  the USB mass-storage driver (M4).
- Class-level error code space (`BLK_ERR_*`) final assignment awaits the core
  `DwpStatus` decision (DWP §13).

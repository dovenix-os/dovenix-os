# Driver Class: `net`

> **Status: draft v0.1.** Second DWP class, following the structure templated by
> [`block`](block.md). Where `block`'s central contract is durability, `net`'s is
> **integrity under loss**: frames may be dropped, but never corrupted,
> duplicated by the driver, or reordered within a queue.

## 1. Scope and roles

Ethernet-semantics packet devices: virtio-net (M2), then real NICs (M4 — e1000e
class hardware, whether native, ported, or VM-backed, all behind this same class).

- **Driver**: exposes one network interface per service channel session.
- **Client**: `netd` (the network stack — smoltcp-lineage per roadmap M3); also
  test harnesses and, later, virtualization backends bridging guest traffic.
- Out of scope for v0: multi-queue/RSS (layout ready, §9), LRO, PTP timestamping,
  Wake-on-LAN (needs a powerd story), MACsec/offloaded crypto.

## 2. Class identity

| | |
|---|---|
| Class name | `net` |
| Class version | `0.1` (negotiated in `HELLO` per DWP §7) |
| `msg_type` namespace | `0x0002_xxxx` |

## 3. Control plane

### 3.1 `NET_GET_INFO` (client → driver)

```rust
struct NetInfo {
    mac: [u8; 6],
    _pad: u16,
    mtu_max: u32,          // largest supported MTU (excl. Ethernet header)
    features: u64,         // bitmap, §3.2
    max_queue_pairs: u16,  // v0 clients use 1
    max_queue_depth: u16,
    link: NetLinkState,    // current state; updates arrive via NET_EV_LINK
}

struct NetLinkState {
    up: u8,
    duplex: u8,            // 0 half, 1 full, 0xff unknown
    _pad: u16,
    speed_mbps: u32,       // 0 = unknown
}
```

### 3.2 Feature bits

| Bit | Feature | Meaning |
|---|---|---|
| 0 | `CSUM_TX` | Driver/device computes L4 checksums on transmit |
| 1 | `CSUM_RX` | Device validates L4 checksums on receive (result in completion flags) |
| 2 | `TSO4` / 3 | `TSO6` | TCP segmentation offload (v4/v6) |
| 4 | `MULTICAST_FILTER` | Hardware multicast filtering (`NET_SET_RX_MODE` list honored in hw, not just software) |
| 5 | `PROMISC` | Promiscuous mode available |
| 6 | `MAC_SET` | MAC address is changeable |

`NET_SET_FEATURES(requested) → enabled`, before queue creation, as in `block`.

### 3.3 Configuration

- `NET_SET_MTU { mtu: u32 }` — driver reconfigures; frames exceeding MTU (+
  headers, + TSO aggregate limits) are rejected per-descriptor, not per-session.
- `NET_SET_MAC { mac: [u8; 6] }` — requires `MAC_SET`.
- `NET_SET_RX_MODE { promiscuous: u8, all_multicast: u8, multicast_count: u16,
  multicast[..]: [u8; 6]... }` — replaces the whole filter state atomically
  (idempotent by construction; matters for restart replay, §6).

### 3.4 Queues

`NET_QUEUE_CREATE` mirrors `block` (§3.3 there) with one addition:

```rust
struct NetQueueCreate {
    queue_id: u16,       // v0: must be 0
    direction: u16,      // 0 = TX, 1 = RX
    depth: u16,
    data_vmo_count: u16,
}
// handles: ring VMO, submit-doorbell Event, complete-doorbell Event, data VMOs
```

A functioning interface needs one TX and one RX queue. Both use the same
submission-ring + completion-ring machinery as `block` — the *meaning* differs:

- **TX**: submission = filled frames to transmit; completion = frame sent (buffer
  reusable).
- **RX**: submission = **empty buffers posted** by the client; completion =
  buffer filled with a received frame (length + flags). The client's job is to
  keep the RX ring stocked; an empty RX ring means drops (§5), never blocking.

### 3.5 Events (driver → client, `txid = 0`)

| Event | Payload |
|---|---|
| `NET_EV_LINK` | `NetLinkState` — link up/down, speed/duplex changes |

## 4. Data plane

Ring VMO layout is identical in shape to `block` §4 (header page with four
cache-line-separated indices, then submission ring, then completion ring), with
net-specific 32-byte descriptors and 16-byte completions:

```rust
struct NetTxDesc {
    req_id: u64,
    data_vmo: u16,
    flags: u16,          // bit 0: CSUM (use csum_start/offset)
                         // bit 1: TSO  (use gso_size / hdr_len)
    csum_start: u16,     // bytes from frame start; L4 checksum insertion
    csum_offset: u16,
    data_offset: u64,    // byte offset in data VMO; frame is contiguous
    len: u32,            // frame length (or aggregate length when TSO)
    gso_size: u16,       // MSS for TSO
    hdr_len: u8,         // L2+L3+L4 header bytes for TSO
    _reserved: u8,
}

struct NetRxDesc {       // buffer post
    req_id: u64,
    data_vmo: u16,
    _pad: [u16; 3],
    data_offset: u64,
    capacity: u32,       // must fit MTU + headers (+ aggregates if negotiated)
    _reserved: u32,
}

struct NetCompletion {
    req_id: u64,
    status: u16,         // NET_OK or NET_ERR_*
    flags: u16,          // RX: bit 0 CSUM_VALID, bit 1 CSUM_CHECKED
    len: u32,            // RX: received frame length; TX: 0
}
```

Rules (deltas from `block` §4 — everything not restated carries over, including
adversarial validation and index-comparison-as-ground-truth):

- **Single contiguous segment per frame** in v0, including TSO aggregates (so RX
  buffers for TSO-negotiated sessions must be sized for aggregates; the
  simplicity/copy trade-off is the same bet as `block`, revisited with M4
  numbers).
- **Ordering**: frames submitted on one TX queue reach the wire in submission
  order. RX completions are delivered in reception order. No cross-queue
  ordering exists (relevant once multi-queue lands).
- **Overload behavior is drop, never block**: RX with no posted buffers drops at
  the device; a full TX ring is client backpressure (submit fails locally —
  the client waits on the completion doorbell). The driver never buffers frames
  in unbounded internal memory.

## 5. Delivery contract

Normative, the analogue of `block` §5 — a network is lossy, so the guarantees
are about *integrity*, not delivery:

1. A frame may be dropped anywhere (device overrun, unposted RX, quiesce window,
   driver restart). Upper layers (TCP, QUIC, application retries) own recovery —
   the class does not pretend otherwise.
2. A completed RX frame is **bit-exact** as received, with `len` correct and
   checksum flags truthful (`CSUM_VALID` set only if actually verified).
3. The driver never **duplicates** a frame (TX: one submission = at most one
   wire transmission; RX: one received frame = at most one completion).
4. The driver never **reorders** within a queue (per §4).
5. TX completion means "handed to the device / on the wire", **not** "delivered".
   It is a buffer-reuse signal, nothing more.

## 6. Crash-restart and retry semantics (per DWP §6.3)

On `ERR_PEER_RESET`:

| Operation | Retry class | Rationale |
|---|---|---|
| TX frames in flight | **Do not retry** (class: *lossy*) | May or may not have hit the wire; retrying risks duplication, which violates §5.3 in spirit. Loss is the contract; TCP/QUIC recover. |
| RX buffers posted | **Re-post freely** | Posting is idempotent; frames that arrived during the gap are simply lost. |
| `NET_SET_FEATURES` / `SET_MTU` / `SET_MAC` / `SET_RX_MODE` | **Safe to replay** | All are absolute-state-setting (never incremental), by design — §3.3. |
| `NET_GET_INFO` | **Safe to retry** | Read-only. |

**The restart rule**: after `ERR_PEER_RESET`, the client replays its
configuration (features → MTU → MAC → RX mode → queues), re-posts RX buffers,
and resumes. Everything in the config plane is deliberately *idempotent
absolute state* so this replay needs no reconciliation logic. The traffic gap is
indistinguishable from a brief cable pull — which the stack must tolerate anyway.

## 7. Quiesce, state, restore (per DWP §6.2)

- **`QUIESCE`**: stop RX (arriving frames drop — permitted by §5.1), complete or
  discard in-flight TX (discarded TX completes with `NET_ERR_CANCELED`; the
  frame counts as dropped), report `QUIESCED`. Unlike `block` there is nothing
  to flush — quiesce is fast and bounded by a single TX drain.
- **State blob**: negotiated version + features, MTU, MAC (if overridden), RX
  mode/filter table, queue configs. Link state is *not* serialized — it is
  re-derived from hardware at restore and re-announced via `NET_EV_LINK`.
- **Restore**: re-map re-granted VMOs, reprogram device from blob (features,
  MTU, MAC, filters), validate rings, resume. The observable effect of a live
  update is a bounded traffic gap — the conformance suite (§8) puts a number on
  "bounded" and enforces zero corruption/duplication across it.

## 8. Conformance suite (definition of "done")

1. **Config plane**: feature negotiation subsets, MTU edges, RX-mode replacement
   atomicity, rejection of malformed/oversized descriptors without crash.
2. **Integrity**: bidirectional traffic soak with per-frame checksums verified
   end-to-end; asserts §5.2–5.4 (bit-exact, no dup, no intra-queue reorder) at
   line rate and under doorbell storms.
3. **Offload truthfulness**: TX csum/TSO output validated against software
   computation; RX `CSUM_VALID` cross-checked against deliberately corrupted
   frames.
4. **Overload**: unposted-RX and full-TX-ring behavior — drops and backpressure
   only; driver memory bounded; no hang.
5. **Crash-restart under traffic**: randomized kills; asserts the §6 replay rule
   suffices, loss-only semantics (every frame accounted as delivered-once or
   dropped, never duplicated/corrupted), stack (netd) recovers TCP sessions.
6. **Live update under traffic**: measures the gap, enforces the bound, asserts
   config survives via blob and link state re-announces.
7. **Adversarial rings**: as `block` §8.4.

## 9. Open questions (v0.1 → v0.2)

- **Multi-queue + RSS**: `queue_id`/`direction` already shape the API; needs
  hash/steering configuration messages and per-queue quiesce semantics. Gated on
  M4 real-NIC contention data (same policy as `block`).
- **Buffer-pool ergonomics**: RX for TSO wants mixed buffer sizes; possibly a
  small/large dual-pool convention rather than uniform `capacity`.
- **Zero-copy fastpath** for netd-bypass workloads (XDP/AF_XDP role): likely a
  separate consumer of the same rings rather than a class change.
- **Wake-on-LAN** and runtime power states: specified together with the
  `powerd` integration (suspend rides DWP quiesce; WoL is an armed-sleep mode).
- Virtio-net descriptor mapping notes (how these descriptors project onto
  virtio's) belong in the M2 driver's docs, not this spec.

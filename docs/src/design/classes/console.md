# Driver Class: `console`

> **Status: draft v0.1.** Fourth DWP class. It is the first **reliable ordered
> stream**: unlike [`net`](net.md), loss is *not* contractual — bytes are delivered in
> order, and the only permitted loss (hardware overrun, restart gap) must be
> detected and announced. This class carries the system's earliest and most
> important I/O: the boot console, and terminals for M1–M3 development and e2e
> harnesses.

## 1. Scope and roles

Bidirectional byte-stream devices: UARTs/serial (M1 — the first driver ever
written for Dovenix), virtio-console (M2), later USB serial.

- **Driver**: one stream per session (a multi-port device exposes one session
  per port via devmgr).
- **Client**: the console/log service, e2e test harnesses, `posixd` (as the
  backend for a real terminal; ptys themselves live in [`posixd`](../architecture.md#posix-strategy-first-class-ports-without-unnecessary-change), not here —
  this class is transport to hardware, line discipline is client policy).
- Out of scope v0: modem-control-driven protocols (PPP-era), RS-485 modes.

## 2. Class identity

| | |
|---|---|
| Class name | `console` |
| Class version | `0.1` |
| `msg_type` namespace | `0x0004_xxxx` |

## 3. Control plane

### 3.1 `CON_GET_INFO`

```rust
struct ConInfo {
    features: u32,   // bitmap, §3.2
    _pad: u32,
    size: ConSize,   // current terminal size; zeros if SIZE unsupported
}

struct ConSize { cols: u16, rows: u16 }
```

### 3.2 Feature bits

| Bit | Feature | Meaning |
|---|---|---|
| 0 | `PARAMS` | UART line parameters settable (§3.3) |
| 1 | `BREAK` | `CON_SEND_BREAK` supported |
| 2 | `SIZE` | Terminal size known; `CON_EV_RESIZE` emitted (virtio-console) |
| 3 | `FLOW_HW` | RTS/CTS hardware flow control available |

### 3.3 `CON_SET_PARAMS` (requires `PARAMS`)

```rust
struct ConParams {
    baud: u32,
    data_bits: u8,   // 5..8
    parity: u8,      // 0 none, 1 odd, 2 even
    stop_bits: u8,   // 1 or 2
    flow: u8,        // 0 none, 1 RTS/CTS (requires FLOW_HW)
}
```

Absolute state, idempotent, blindly replayable (the pattern set by
[`net` §3.3](net.md#33-configuration)).

### 3.4 `CON_SEND_BREAK { duration_ms: u16 }` (requires `BREAK`)

### 3.5 Events (driver → client)

| Event | Payload | Meaning |
|---|---|---|
| `CON_EV_RESIZE` | `ConSize` | Terminal size changed (`SIZE` only) |
| `CON_EV_BREAK` | — | Break condition received on the line |

## 4. Data plane

Two generic queues ([DWP core §8](../driver-wire-protocol.md#8-data-plane-generic-queues)),
shaped exactly like `net`'s TX/RX — a `direction` field in `CON_QUEUE_CREATE`
mirrors [`NET_QUEUE_CREATE`](net.md#34-queues) — but
entries describe **byte chunks, not frames**: chunk boundaries carry no
meaning, and the driver may split or coalesce freely in either direction.

```rust
struct ConChunk {         // S = 32; TX: filled bytes; RX: empty buffer posted
    req_id: u64,
    data_vmo: u16,
    _pad: [u16; 3],
    data_offset: u64,
    len: u32,             // TX: bytes to send; RX: buffer capacity
    _reserved: u32,
}

struct ConCompletion {    // C = 16
    req_id: u64,
    status: u16,          // CON_OK or CON_ERR_*
    flags: u16,           // RX bit 0: OVERRUN occurred before this chunk
    len: u32,             // TX: bytes accepted (== len); RX: bytes received
}
```

- TX completion means the bytes are **handed to the device FIFO** (buffer
  reusable), not that the far end read them — same honesty rule as `net`'s TX.
- Small queue depths are expected (serial is slow); virtio-console sessions may
  negotiate deeper queues. Nothing in the class changes with rate.

## 5. Stream contract

Normative — this is the reliable-ordered counterpoint to `net` §5:

1. Bytes are delivered **in order, both directions**, with no reordering, no
   duplication, and no invention, ever.
2. **Loss is exceptional, bounded, and announced**: permitted only on hardware
   receive overrun (unposted RX while the UART FIFO fills) or across a restart
   gap, and always surfaced (`OVERRUN` flag / restart epoch). Silent loss is a
   conformance failure.
3. Flow control: with `FLOW_HW` negotiated, the driver asserts backpressure
   instead of overrunning; without it, §5.2 is the client's incentive to keep
   RX posted.
4. TX is never dropped by the driver: a full TX queue is client backpressure;
   accepted TX chunks are sent unless the session dies.

## 6. Crash-restart and retry semantics (per DWP §6.3)

| Operation | Retry class | Rationale |
|---|---|---|
| TX chunks in flight | **Do not retry** | Unknown how much reached the wire; re-sending duplicates bytes into an ordered stream — worse than the gap. The gap is announced; stream users (shells, harnesses) tolerate line noise. |
| RX buffers posted | **Re-post freely** | Posting is contentless |
| `CON_SET_PARAMS` | **Safe to replay** | Absolute state |
| `CON_SEND_BREAK` | **Do not retry** | Duplicate breaks are observable line events |
| `CON_GET_INFO` | **Safe to retry** | Read-only |

**Restart rule**: replay params, re-post RX, resume. The client treats the gap
as line noise on RX and an unknown-length tail loss on TX — protocols that
need better (file transfer over serial) already checksum above the stream.

## 7. Quiesce, state, restore

- **[`QUIESCE`](../driver-wire-protocol.md#62-quiesce--serialize--restore-live-update)**:
  drain the TX FIFO to the wire (bounded by baud rate — the
  deadline must account for it), stop RX into the device FIFO, report.
- **State blob**: negotiated version + features + current `ConParams`.
  Terminal size is *not* serialized — re-queried/re-announced at restore
  (the [`net` link-state precedent](net.md#7-quiesce-state-restore-per-dwp-62)).
- **Restore**: reprogram params, resume; RX bytes that arrived during the gap
  are lost and flagged `OVERRUN` per §5.2.

## 8. Conformance suite

1. **Order & integrity**: bidirectional soak with pattern verification —
   asserts §5.1 exactly (no reorder/dup/invention) across chunk-size sweeps
   and split/coalesce behavior.
2. **Params**: line-setting matrix against a loopback peer; replay idempotence.
3. **Overrun**: starve RX under incoming flood; assert bounded, announced loss
   only; with `FLOW_HW`, assert zero loss and real backpressure.
4. **Restart mid-stream**: randomized kills during bidirectional transfer;
   assert announced gap, ordered delivery on both sides of it, params replay
   suffices.
5. **Quiesce under load**: TX drain completes within baud-derived deadline;
   live update presents as a clean announced gap.
6. **Adversarial rings** per [core §8.5](../driver-wire-protocol.md#85-trust-and-validation).

## 9. Open questions (v0.1 → v0.2)

- Modem control lines (DTR/DSR/DCD/RI): feature bit + event + set-message when
  a real use case lands (likely with USB serial in M4).
- Whether the kernel's *earliest* boot console (before devmgr exists) is a
  degenerate in-kernel writer of this format or a separate mechanism (lean:
  separate — pre-userspace logging is a kernel doc concern, and this class
  starts at devmgr time).
- Log-structured console multiplexing (many readers of one boot log) — client
  service design, not class protocol, but the buffer story should be sketched
  before M1's harness hardcodes something.

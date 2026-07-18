# Driver Class: `input`

> **Status: draft v0.1.** Third DWP class. It demonstrates two firsts: a pure
> driver→client event stream on the generic queue machinery, and the **inline
> credit model** — no data VMOs, events ride inside completion entries. It is
> also the simplest class, deliberately: near-stateless, trivial quiesce.

## 1. Scope and roles

Human input event devices with evdev-inspired semantics: keyboards, mice,
touchpads/touchscreens, tablets, gamepads. First implementation: virtio-input
(M2); then USB HID and laptop internals (M4).

- **Driver**: one input device per session (a combo keyboard+pointer device
  exposes multiple devices via devmgr, one session each).
- **Client**: the input server / compositor (later: seatd-role component).
  Keymaps, key repeat, pointer acceleration, and gestures are **client policy**
  — the driver reports raw events only (matching the evdev/libinput split).
- Out of scope v0: force feedback, absolute-device calibration protocol.

## 2. Class identity

| | |
|---|---|
| Class name | `input` |
| Class version | `0.1` |
| `msg_type` namespace | `0x0003_xxxx` |

## 3. Control plane

### 3.1 `INPUT_GET_INFO`

```rust
struct InputInfo {
    kinds: u32,          // bitmap: KEYBOARD, POINTER, TOUCH, TABLET, GAMEPAD
    vendor: u16, product: u16, version: u16,
    _pad: u16,
    name: [u8; 64],      // UTF-8, NUL-padded
}
```

### 3.2 Capability and state queries

- `INPUT_GET_CAPS { ev_type: u16 } → bitmap` — which event codes the device can
  emit for an event type (evdev-style type/code space, §4.1).
- `INPUT_GET_STATE { ev_type: u16 } → bitmap` — codes **currently asserted**
  (keys held, buttons down, switch positions). Exists specifically so clients
  can resynchronize after event loss or driver restart (§6) — the stuck-modifier
  bug is designed out, not patched around.

### 3.3 `INPUT_SET_LED { code: u16, on: u8 }`

The only mutating config in v0 (caps-lock LED and friends). Absolute state →
idempotent → blindly replayable.

## 4. Data plane: the inline credit model

One generic queue ([DWP core §8](../driver-wire-protocol.md#8-data-plane-generic-queues)),
**no data VMOs**. A submission is a bare
credit; the driver spends one credit per event, delivering the event **inline
in the completion entry**:

```rust
struct InputCredit {      // S = 16
    req_id: u64,
    _reserved: u64,
}

struct InputEvent {       // C = 32
    req_id: u64,
    status: u16,          // INPUT_OK
    flags: u16,           // bit 0: OVERRUN — events were dropped before this one
    ev_type: u16,         // KEY, REL, ABS, SW, SYN, ...
    code: u16,            // e.g. KEY_A, REL_X, ABS_MT_POSITION_X
    value: i32,
    timestamp_ns: u64,    // kernel monotonic clock, stamped at interrupt time
    _pad: u32,
}
```

### 4.1 Event semantics (evdev-inspired)

- The type/code/value vocabulary follows evdev's model (KEY/REL/ABS/SW plus
  SYN framing; multitouch via ABS_MT slots). The exact code table is defined in
  [`dwp-schema`](../driver-wire-protocol.md#10-idl-and-code-generation) —
  evdev-*inspired*, not ABI-identical (we are not bound by Linux header values,
  and [clean-room rules](../licensing.md#operational-rules) apply).
- Events between two `SYN_REPORT`s form one device report and share a
  timestamp; clients coalesce at SYN boundaries, exactly like evdev consumers.

### 4.2 Overflow

If the credit ring runs dry, the driver drops **whole reports** (never a torn
report) and sets `OVERRUN` on the next delivered event. On seeing `OVERRUN`,
the client resynchronizes via `INPUT_GET_STATE` — the evdev `SYN_DROPPED`
discipline, made mandatory.

## 5. Delivery contract

1. Events are delivered in device order; reports are never torn (§4.2).
2. Loss is permitted only under credit exhaustion or restart, is always
   *report-granular*, and is always **announced** (`OVERRUN` flag / restart
   epoch) — silent loss is a conformance failure.
3. Timestamps are monotonic and stamped at hardware interrupt time, not at
   delivery time (input latency must be measurable, so it must not be hidden).

## 6. Crash-restart and retry semantics (per DWP §6.3)

| Operation | Retry class | Rationale |
|---|---|---|
| Posted credits | **Re-post freely** | Credits are contentless |
| `INPUT_SET_LED` | **Safe to replay** | Absolute state |
| `GET_INFO` / `GET_CAPS` / `GET_STATE` | **Safe to retry** | Read-only |

**Restart rule**: re-post credits, replay LED state, then `INPUT_GET_STATE` to
resynchronize pressed keys/buttons — identical to the `OVERRUN` path. A driver
restart is, to the client, just a large announced overflow.

## 7. Quiesce, state, restore

- **`QUIESCE`**: stop event delivery, discard the device's internal event FIFO
  (input events are transient by nature — §5.2 covers the loss). Immediate.
- **State blob**: negotiated version + LED state. This class is the
  degenerate-blob case and serves as the minimal example of
  [DWP §6.4](../driver-wire-protocol.md#64-state-blobs).
- **Restore**: reprogram LEDs, resume. Clients observe it as an `OVERRUN`.

## 8. Conformance suite

1. **Vocabulary**: caps bitmaps are truthful (every advertised code observable,
   no unadvertised codes emitted); report framing always SYN-terminated.
2. **Ordering & framing**: high-rate synthetic event storms (8 kHz-mouse-class)
   arrive in order, untorn, timestamps monotonic.
3. **Overflow**: starve credits under storm; assert report-granular drops,
   `OVERRUN` announced, `GET_STATE` resync yields truth.
4. **Restart**: kill mid-storm with keys held; assert the client resync path
   ends with correct key state — the stuck-modifier test, automated.
5. **Adversarial**: hostile credit ring per
   [core §8.5](../driver-wire-protocol.md#85-trust-and-validation).

## 9. Open questions (v0.1 → v0.2)

- Force feedback / haptics (needs a client→driver data direction: second queue).
- Whether high-resolution scroll and touch pressure need dedicated types or
  follow evdev's REL_WHEEL_HI_RES / ABS_PRESSURE precedent (lean: precedent).
- Event-code table governance in `dwp-schema` — versioning discipline for adding
  codes (likely: class minor bump per addition batch).

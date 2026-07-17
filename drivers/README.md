# drivers

Device drivers. Every driver is an isolated process speaking the
[Driver Wire Protocol](../docs/src/design/driver-wire-protocol.md).

## `native/` — `Apache-2.0 OR MIT`

Drivers written from hardware specs: virtio family first (M2), then NVMe, xHCI,
AHCI (M4).

## `ported/` — `GPL-2.0-only` islands

Drivers ported from Linux source. **This is the only directory in the repository
where GPL-derived code may exist.** One subdirectory per driver, each with its own
`LICENSE` and `PROVENANCE.md` (upstream repo, kernel version, commit). Rules in
[CONTRIBUTING.md](../CONTRIBUTING.md) and the
[Licensing Model](../docs/src/design/licensing.md).

- Status: empty (first ports expected after milestone M4, only where the Linux
  driver VM hop hurts).

# libs

Everything linkable: DWP schema/codecs, driver SDK, IPC runtime, the clean-room
Linux shim.

- License scope: **strictly `Apache-2.0 OR MIT`** — these crates are linked by
  GPL-2.0-only driver islands (under MIT), so they must never gain GPL-incompatible
  code or dependencies. See [Licensing Model](../docs/src/design/licensing.md).
- The `linux-shim` crate additionally follows clean-room discipline: written from
  documentation only, never while reading GPL implementation source.
- Status: not started (first crates land with milestone M2: `dwp-schema`, `dwp`,
  `driver-sdk`).

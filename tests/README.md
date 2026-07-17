# tests

End-to-end and integration suites — the only kind of tests Dovenix has, per
[goal 5](../docs/src/vision/goals.md) and the
[Testing Strategy](../docs/src/dev/testing.md).

- `conformance/` — per-protocol-class harnesses (a driver is "done" when it passes)
- `scenario/` — partial-system compositions under workloads and fault injection
- `system/` — full boots, live-update/rollback drills, Linux-baseline benchmarks

License scope: `Apache-2.0 OR MIT`. Status: not started (first harness lands with
milestone M1).

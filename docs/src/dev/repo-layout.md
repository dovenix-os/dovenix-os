# Repository Layout

Dovenix is a **monorepo, structured for later fission**: every top-level directory
is an independently buildable, independently licensable unit that could be split
into its own repository without surgery. The enforcement mechanism is simple —
components may only depend on `libs/`, never on each other's internals; all
cross-component contact happens over protocols.

```text
dovenix-os/
├── LICENSE-APACHE, LICENSE-MIT     # the dual license
├── README.md, CONTRIBUTING.md
├── Cargo.toml                      # workspace root (members added as crates land)
├── rust-toolchain.toml             # pinned toolchain
├── docs/                           # this book (mdBook) — normative design docs
│   ├── book.toml
│   └── src/
├── plans/                          # implementation plans (working docs, not in the book)
├── kernel/                         # the Dovenix kernel (no_std Rust)
├── libs/                           # everything linkable — MUST stay Apache-2.0 OR MIT
│   ├── dwp-schema/                 # (future) wire-protocol IDL: source of truth
│   ├── dwp/                        # (future) generated codecs + session layer
│   ├── driver-sdk/                 # (future) driver-host runtime
│   └── linux-shim/                 # (future) clean-room Linux API layer for ports
├── servers/                        # system servers: devmgr, vfsd, netd, posixd, …
├── drivers/
│   ├── native/                     # Apache-2.0 OR MIT drivers (virtio, nvme, xhci…)
│   └── ported/                     # GPL-2.0-only islands, one dir per driver
├── tools/                          # host tooling: image builder, test runner, CI glue
└── tests/                          # e2e & integration suites, conformance harnesses
```

## Rules that keep it modular

1. **Dependency direction**: `kernel` depends on nothing in-tree; `servers/`,
   `drivers/`, `tools/`, `tests/` depend only on `libs/`. No exceptions.
2. **License scoping**: `drivers/ported/` is the only place GPL code may exist.
   `libs/` must remain linkable from GPL islands (`Apache-2.0 OR MIT`, no
   GPL-incompatible transitive deps).
3. **One Cargo workspace** at the root for now; each directory keeps its own crate
   graph coherent so it can become a standalone workspace when split out.
4. **Tests live with the system, not the component**: per goal 5, components are
   tested end-to-end from `tests/` via their protocols, not with in-crate unit
   tests.
5. **Book vs. plans**: this book is normative design (what and why); `plans/`
   holds implementation plans (how and in what order) as working documents for
   contributors and agents — see `plans/README.md`. When they disagree, the
   book wins.

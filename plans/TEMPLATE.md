---
status: draft            # draft | active | done | superseded
updated: YYYY-MM-DD
depends-on: []           # other plans, e.g. [m1-boot]
approved-by: pending     # filled in ONLY by the approving human: "name, YYYY-MM-DD"
---

# <Milestone or feature name>

## Goal

<One paragraph: the demo this plan delivers, stated as an observable outcome.
For milestones, quote the roadmap entry.>

## Non-goals (for this plan)

- <Deliberate deferrals, with where they went>

## Order of work

<Every phase ends at a human checkpoint: run the phase-completion protocol in
CLAUDE.md (audit — reliability and Rust idioms first, security last, on the
final code → update docs and docstrings and confirm them → pause for human
review). Do not start the next phase unreviewed.>

### Phase 1 — <name>

- [ ] <Task: one reviewable change, with its verifiable outcome>
- [ ] …

### Phase 2 — <name>

- [ ] …

## Verification

<The e2e tests / conformance runs that define done for this plan. Per goal 5
there are no unit tests — every task's outcome must be observable from a
booted (partial) system or a host-side tool run.>

## Deviations

<Append-only. "YYYY-MM-DD: planned X, did Y because Z.">

## Open items

- <Unresolved questions that don't block starting>

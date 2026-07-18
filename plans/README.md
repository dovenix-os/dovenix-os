# plans/

Implementation plans for milestones and features. **Working documents, not the
book**: the [Dovenix Book](../docs/) is normative and public-facing (what the
system is and why); plans are the operational guide for whoever — human or
agent — is doing the work (how, in what order, what's done, what's blocked).

## Rules

1. **The book wins.** If a plan contradicts a design doc, the design doc is
   right; either follow it or change it first (in the same PR, with the
   version-bump discipline the doc requires). Plans never introduce design —
   they sequence it.
2. **One plan per milestone** (`m1-boot.md`, `m2-virtio.md`, …) and **one per
   nontrivial cross-cutting feature** (`feature-<name>.md`). Small features
   don't need plans.
3. **Every plan carries frontmatter**: `status` (draft | active | done |
   superseded), `updated` (date), `depends-on`, and `approved-by` — the human
   who approved the plan (name + date), filled in **only by that human**.
   Exactly **one** milestone plan is `active` at a time, and a plan may not be
   `active` (nor may implementation start) while `approved-by` is `pending`.
4. **Phases are human-verifiable and human-gated.** Every phase ends in
   something a human can directly verify, and completing a phase means running
   the phase-completion protocol in `CLAUDE.md` (audit → docs updated and
   confirmed → pause for review). Agents do not roll from one phase into the
   next unreviewed.
5. **Keep plans current.** When a work session changes reality (task done,
   approach changed, blocker found), update the plan in the same commit as the
   code. A stale plan is worse than no plan. Done plans stay (marked `done`) as
   the record of how it actually went; log significant deviations in the
   plan's Deviations section rather than rewriting history.
6. **Task granularity**: a task is one reviewable change (roughly one PR) with
   its e2e-verifiable outcome stated. Not "write the scheduler" — "boot two
   threads that ping-pong over a channel, asserted by the M1 harness".

## Format

Copy [`TEMPLATE.md`](TEMPLATE.md). Sections: Goal (the milestone's demo,
verbatim from the roadmap), Non-goals for this plan, Order of work (phased
task checklists), Verification (which e2e tests gate completion), Deviations
(append-only log), Open items.

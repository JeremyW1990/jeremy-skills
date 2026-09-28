# Project Notes

`.loop-tickets/project-notes.md` is a tracked project dashboard and learning ledger shared
through the pool's single PR. It is derived from authoritative external queue state and
available validation receipts; never use it to resume a queue, replace a ticket or skip
an authorized test, required repository check or mandatory protection.

## Location and delivery

Use the repository-relative path `.loop-tickets/project-notes.md` in the pool branch's
worktree. Never write the main worktree from a linked pool worktree. If the path contains
an unrelated file or an incompatible schema, preserve it and block delivery for user
direction. Existing `loop-tickets-project-notes/v1` notes can be upgraded to the v2 shape
below, preserving useful observations and previously merged history.

After implementation, authorized focused tests (or an explicit unrun record) and optional
review, write and stage the note with the ticket's final local commit. During the pool,
use `cumulative-on-commit`: project the current ticket as `committed` if this commit
succeeds, retain earlier committed tickets and derive the next frontier. State explicitly
that this pool remains unmerged. Do not claim the pending hook has already passed;
record its actual outcome afterward only in external state.

For the last ticket's final commit, use `effective-on-merge` instead: project all pool
tickets as `done` conditional on the one final PR merging. This is a labeled future
projection, not a claim that delivery has already happened. Reconcile it during final
integration if target movement or repairs change its assumptions.

Stage only the exact note path with the implementation or integration commit. If an old
local ignore rule hides the initial file, use
`git add -f -- .loop-tickets/project-notes.md`; do not change `.gitignore` or stage the
whole directory. Never create a note-only branch, commit, push, PR, review, test or CI run.
Keep transient check outcomes, PR URLs, merge SHAs and merge-action results in external
state because backfilling them would require another push.

Use this compact shape:

```markdown
---
schema: loop-tickets-project-notes/v2
repository: <canonical identity>
updated_at: <ISO-8601>
snapshot: cumulative-on-commit | effective-on-merge
pool: <pool ID>
ticket: <current or final ticket ID>
---

> Advisory and derived; current sources, instructions, code and pool state win.

## Current pools
| Pool/source version | Snapshot progress | Delivery state | Next frontier | Blocked | Next action |
|---|---|---|---|---|---|

## Queue focus
| Pool | Ticket | Title | Snapshot state | Next action or blocker |
|---|---|---|---|---|

## Validation economics
| Pool | Runs/failures/reused | Recorded duration | Largest current cost |
|---|---|---|---|

## Reusable observations
| ID | Status/kind | Scope | Finding -> next action | Impact | Evidence/revalidate when |
|---|---|---|---|---|---|
```

Show counts plus only the active, frontier, blocked and recently committed or merged
tickets. Distinguish committed-but-unmerged progress from merged history. In the final
snapshot, label completion as effective only on merge and project no remaining frontier
for this pool. Actual pending integration stays in external state. Collapse older
delivered pools to one recent-summary row; external pool state holds the full history.
Neither snapshot invents a future result or merge ID.

Give each observation a stable ID and deduplicate repeated patterns by that ID, increasing
an occurrence count instead of appending variants. Allowed statuses are `candidate`,
`verified`, `needs-revalidation` and `superseded`; useful kinds include `fast-path`,
`failure-pattern`, `optimization` and `ticket-design`.

Keep the finding and next action together. Label impact as measured or estimated, cite a
compact pool/ticket/receipt pointer, and name the fingerprints or condition that require
revalidation. Mark a verified row `needs-revalidation` when those inputs change. A
candidate helps locate what to inspect but cannot justify evidence reuse or omitted work.
Required checks and current project instructions remain global guardrails for every row.

Retain at most 20 active observations, prioritizing recurring or highest-impact findings.
Remove superseded detail after its replacement is recorded. Never copy ticket bodies,
code, full command output, absolute local paths, private tracker content, secrets,
environment values, customer data or fixture payloads into this GitHub-shared file.

Start from the target version captured at pool start and accumulate later ticket updates
on the same branch. On resume, keep that accumulated note. At final integration, merge
rows from the newly fetched target deterministically by pool ID and observation ID;
preserve pool progress and reconcile stale assumptions. Include any correction with the
integration or repair commit, never a note-only follow-up. Keep temporary files outside
the tracked directory and stage only `.loop-tickets/project-notes.md`. A generation,
schema or staging failure blocks the associated commit unless the user waives the note.

Keep actual test execution status, required repository check results and merge results
in external state. Observe checks at the points configured by the repository; earlier
results may become stale during repairs. A later implementation commit may include
already-known metrics, but never create a commit or push solely to backfill them.

`ticket-design` observations can flag over-fragmentation, hidden dependencies, repeated
end-to-end setup or poor validation-to-implementation ratios. They are human guidance and
input to later loop runs. Do not claim that `to-tickets` consumes them unless that skill
has separately been updated to do so.

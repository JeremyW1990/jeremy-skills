# Project Notes

`.loop-tickets/project-notes.md` is a tracked project dashboard and learning ledger shared
through GitHub. It is derived from authoritative external queue state and focused-test
receipts; never use it to resume a queue, replace a ticket or skip a current-ticket test
or mandatory protection.

## Location and delivery

Use the repository-relative path `.loop-tickets/project-notes.md` in the current ticket
branch's worktree. Never write the main worktree from a linked ticket worktree. If the
path contains an unrelated file or an incompatible schema, preserve it and block delivery
for user direction.

After implementation, focused tests and optional review, verify the base is stable, then
write and stage the final note once. It is an effective-on-merge projection: show the
current ticket as `done` and derive the state that results if this change merges. Stage
only the exact note path with the ticket's final local commit before its first push. If
an old local ignore rule hides the initial file, use
`git add -f -- .loop-tickets/project-notes.md`; do not change `.gitignore` or stage the
whole directory.

Never create a note-only branch, commit, push, PR, review, test or CI run. The projection
reaches the base branch only if the implementation PR merges, at which point it is true.
Keep transient statuses, PR URLs, merge SHAs and current remote-CI results in external
state because recording them would require another push.

Use this compact shape:

```markdown
---
schema: loop-tickets-project-notes/v1
repository: <canonical identity>
updated_at: <ISO-8601>
snapshot: effective-on-merge
ticket: <current ticket ID>
---

> Advisory and derived; current sources, instructions, code and pool state win.

## Current pools
| Pool/source version | Projected progress | Next frontier | Blocked | Next action |
|---|---|---|---|---|

## Queue focus
| Pool | Ticket | Title | Projected state | Next action or blocker |
|---|---|---|---|---|

## Validation economics
| Pool | Runs/failures/reused | Recorded duration | Largest current cost |
|---|---|---|---|

## Reusable observations
| ID | Status/kind | Scope | Finding -> next action | Impact | Evidence/revalidate when |
|---|---|---|---|---|---|
```

Show queue counts plus only active, frontier, blocked and recently completed tickets.
Project the current ticket as completed and derive the resulting frontier. Collapse
drained pools to one recent-summary row; external pool state holds the full history.

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

Start from the synchronized base version and merge rows deterministically by pool ID and
observation ID. If the base changes after a local draft but before the first push, discard
that unpushed draft, synchronize, rerun only invalidated current-ticket tests and stage
the replacement; only the final pushed version counts as the ticket's note update. Keep
temporary files outside the tracked directory and stage only
`.loop-tickets/project-notes.md`. A generation, schema or staging failure blocks the
ticket's push unless the user waives the note.

The current ticket's mandatory hook/CI results cannot appear in its projection because
they happen after the single push. A later ticket may add already-known remote metrics,
but never create a push solely to backfill them.

`ticket-design` observations can flag over-fragmentation, hidden dependencies, repeated
end-to-end setup or poor validation-to-implementation ratios. They are human guidance and
input to later loop runs. Do not claim that `to-tickets` consumes them unless that skill
has separately been updated to do so.

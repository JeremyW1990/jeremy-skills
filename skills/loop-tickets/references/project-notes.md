# Project Notes

`PROJECT-NOTES.md` is a host-local, per-repository dashboard and learning ledger. It is
derived from authoritative pool state and validation receipts; never use it to resume a
queue, replace a ticket, or skip a currently required check.

Use this compact shape:

```markdown
---
schema: loop-tickets-project-notes/v1
repository: <canonical identity>
updated_at: <ISO-8601>
---

> Advisory and derived; current sources, instructions, code and pool state win.

## Current pools
| Pool/source version | Progress | Active/frontier | Blocked | Next action |
|---|---|---|---|---|

## Queue focus
| Pool | Ticket | Title | State | PR | Next action or blocker |
|---|---|---|---|---|---|

## Validation economics
| Pool | Runs/failures/reused | Recorded duration | Largest current cost |
|---|---|---|---|

## Reusable observations
| ID | Status/kind | Scope | Finding -> next action | Impact | Evidence/revalidate when |
|---|---|---|---|---|---|
```

Show queue counts plus only active, frontier, blocked and recently completed tickets.
Collapse drained pools to one recent-summary row; the pool state holds the full history.

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
code, full command output, secrets, environment values, customer data or fixture payloads.

Serialize updates with a repo-key-level lock. While holding it, reread the note, merge by
pool ID and observation ID, then atomically replace it. If rendering fails, leave queue
state untouched and retry at the next normal update boundary.

`ticket-design` observations can flag over-fragmentation, hidden dependencies, repeated
end-to-end setup or poor validation-to-implementation ratios. They are human guidance and
input to later loop runs. Do not claim that `to-tickets` consumes them unless that skill
has separately been updated to do so.

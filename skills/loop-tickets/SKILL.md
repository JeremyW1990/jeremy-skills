---
name: loop-tickets
description: "Work a dependency-ordered pool of implementation tickets to completion, one ticket per branch and PR, with persistent state and deduplicated validation. Use after /to-tickets or whenever the user asks to drain a ticket queue end to end."
---

# Loop Tickets

Work one implementation ticket at a time until the dependency graph is drained.
`/to-tickets` usually produces the pool; this skill owns durable queue state, workflow
selection, validation deduplication, and PR delivery.

## Process

### 1. Capture the pool

`/to-tickets` publishes every ticket it drafts, so read what it wrote this session:
`.scratch/<feature-slug>/issues/*.md`, or the issues it just created. That is the only
source. Do not transcribe tickets from the conversation — prose paraphrases acceptance
criteria, and a silent fall back to it turns a locatable failure into a quality loss
nobody sees.

Found nothing? Name the paths you looked in and stop. Either `/to-tickets` has not run,
or it published somewhere you have not looked.

Copy them into `~/.claude/loop-tickets/<repo>/tickets.json` — id, title, what to build,
acceptance criteria, blocked-by, `status: "todo"`, and source path or URL — before
starting the first ticket. A long run gets compacted, and the tickets are otherwise
unrecoverable. Keep the file outside the repository so the working tree stays clean.

Before implementation, validate the pool against its authoritative tracker and every
`blocked-by` edge:

- Every blocker is in the pool, already closed externally, or recorded as an intentional
  external dependency with its source and reason. A missing unresolved blocker is a queue
  defect; capture it before continuing.
- Record any ticket runner selected by the user, project configuration, or ticket text.
  Default to `/implement` only when none is selected.

Inspect the actual project scripts, hooks, CI, and project instructions and record a small
verification coverage map. A gate subsumes another command only when its current
implementation really invokes that command; never infer coverage from the gate's name.

### 2. Work the ticket frontier

Take the lowest-numbered `todo` implementation ticket whose blockers are all `done`. For
each:

- Branch from the freshly pulled base.
- **Create new test data for this ticket**, tagged with the ticket id so it is
  identifiably yours. Never reset the database, and never reuse or delete an earlier
  ticket's data — reading what is already there is fine and often the point.
- Run the recorded runner in a fresh context — a subagent — so ticket N+1 does not inherit
  ticket N's. Pass the exact ticket source, workflow binding, and verification map.
- While building, run the focused tests needed for red/green feedback. Also run required
  integration, migration, database, and ticket-specific tests that the final gate does not
  cover.
- Before final verification, inspect what the actual final gate, git hooks, and CI commands
  invoke. Run each required validation once on unchanged inputs.
- If the final `git push` invokes a pre-push hook that already runs lint, build, typecheck,
  and static tests, do not manually run those commands first and then repeat them through
  the push. The successful hook output is the evidence for those checks.
- Rerun a covered command separately only when its hook was skipped, the hook's coverage
  changed, failure diagnosis requires it, or the user or ticket explicitly requires it.
- Open the PR and merge it when the resulting verification set is green.
- Mark the ticket `done`, pull the base, and take the next. Without the status write the
  loop re-picks the same ticket.

Honour `max=N` if given. Under `dryRun`, open each PR and stop without merging.

### 3. Stop

Stop when the pool is drained, or when a ticket needs a product decision the tickets never
ruled on. Difficulty is not such a decision — a hard ticket, a failing test, or something
inferable from existing precedent all get worked, not skipped.

To skip one: comment on it saying what decision is needed and what it blocks, leave it
open, and move on. Three consecutive skips, or five in total, and stop and report — at
that point the specification is probably incomplete.

## Rules

- One implementation ticket, one branch, one PR. Combine issue delivery only when project
  configuration or authoritative ticket text explicitly says the same implementation
  satisfies multiple issues; never batch tickets merely because they are adjacent or
  convenient.
- Command coverage is behavioral, not ceremonial: do not run the same lint, build,
  typecheck, test, audit, or push gate twice on unchanged inputs merely to list it twice.
- Never omit an uncovered acceptance check. Gate deduplication removes duplicate execution,
  not evidence.
- Each ticket's test data is its own; the database is never reset between tickets.
- Keep pushes on feature branches.
- Ask when a decision is the user's to make. Never guess on one.

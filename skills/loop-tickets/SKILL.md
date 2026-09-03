---
name: loop-tickets
description: "Work a dependency-ordered ticket pool to completion with persistent state, explicit per-ticket workflow, deduplicated project gates, and PR delivery. Use after /to-tickets or whenever the user asks to drain a ticket queue end to end."
argument-hint: "[runner=<skill>] [max=N] [dryRun]"
disable-model-invocation: false
---

# Loop Tickets

Work one implementation unit at a time until the dependency graph is drained. `/to-tickets`
usually produces the pool; this skill owns durable queue state, workflow selection, gate
deduplication, PR delivery, and administrative closure nodes.

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
acceptance criteria, blocked-by, shape, `status: "todo"`, and source path or URL — before
starting the first ticket. A long run gets compacted, and the tickets are otherwise
unrecoverable. Keep the file outside the repository so the working tree stays clean.

Before implementation, validate the pool against its authoritative tracker and every
`blocked-by` edge:

- Every blocker is in the pool, already closed externally, or recorded as an intentional
  external dependency with its source and reason. A missing unresolved blocker is a queue
  defect; capture it before continuing.
- Classify executable work separately from trackers, epics, and findings. A tracker or
  epic whose body says it is only a closure node gets `execution: "closure"`; do not give
  it a code-change cycle. A finding that explicitly assigns its entire fix to another
  ticket gets `satisfiedBy: <id>` and is closed by that implementation rather than selected
  independently. A finding with any work of its own remains executable.
- Record the user's chosen ticket runner and review policy once. Default to `/implement`
  only when the user has not selected another workflow. If the user chooses `/tdd` with no
  independent review, do not silently reintroduce `/implement` or `/code-review`.

Inspect the actual project scripts, hooks, CI, and project instructions and record a small
verification coverage map. A gate subsumes another command only when its current
implementation really invokes that command; never infer coverage from the gate's name.

### 2. Work the executable frontier

Take the lowest-numbered executable ticket whose blockers are all `done`. Exclude closure
nodes and findings with `satisfiedBy` until their owning work has landed. For each:

- Branch from the freshly pulled base.
- **Create new test data for this ticket**, tagged with the ticket id so it is
  identifiably yours. Never reset the database, and never reuse or delete an earlier
  ticket's data — reading what is already there is fine and often the point.
- Run the recorded runner in a fresh context — a subagent — so ticket N+1 does not inherit
  ticket N's. Pass the exact ticket source, workflow binding, and verification map.
- While building, run focused red/green tests and any targeted check useful for feedback.
  At final verification, run every required guarantee once:
  - Run ticket-specific and database-backed suites not covered by the project gate.
  - If the final gate already runs lint, build, typecheck, or a static test subset, do not
    run those commands separately immediately before the final gate. The final gate's
    successful output is their evidence. Likewise, if pushing invokes that same gate, do
    not manually run it and then run it again through the push.
  - Run generators, artifact rebinds, and final audits after the last relevant edit; rerun
    them only when an input changes.
  - Never describe a skipped, inherited-red, or staged check as green. A project-specific
    staged substitute is valid only when the ticket or user explicitly authorizes it and
    the queue records that authorization.
- Open the PR and merge it when the resulting verification set is green.
- Mark the implementation ticket and every explicitly satisfied finding `done`, pull the
  base, and take the next. Without the status write the loop re-picks the same ticket.

Honour `max=N` if given. Under `dryRun`, open each PR and stop without merging.

### 3. Close administrative nodes

When a tracker or epic reaches the frontier, reconcile its checklist and authoritative
closure conditions. Close or update it without a branch, code PR, synthetic test data, or
development gates unless its own body explicitly requires repository changes. Never count
an administrative closure as an implemented ticket.

### 4. Stop

Stop when the pool is drained, or when a ticket needs a product decision the tickets never
ruled on. Difficulty is not such a decision — a hard ticket, a failing test, or something
inferable from existing precedent all get worked, not skipped.

To skip one: comment on it saying what decision is needed and what it blocks, leave it
open, and move on. Three consecutive skips, or five in total, and stop and report — at
that point the specification is probably incomplete.

## Rules

- One implementation unit, one branch, one PR. It may close multiple issue ids only when
  their authoritative bodies explicitly say the same implementation satisfies them; do
  not batch merely adjacent or convenient tickets.
- Command coverage is behavioral, not ceremonial: do not run the same lint, build,
  typecheck, test, audit, or push gate twice on unchanged inputs merely to list it twice.
- Never omit an uncovered acceptance check. Gate deduplication removes duplicate execution,
  not evidence.
- Each ticket's test data is its own; the database is never reset between tickets.
- Keep pushes on feature branches.
- Ask when a decision is the user's to make. Never guess on one.

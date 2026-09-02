---
name: loop-tickets
description: "Work a pool of tickets to completion, one at a time, running /implement on each. Use when the user wants a set of tickets worked through end to end — after /to-tickets has produced them, or whenever they ask to work the queue, drain the tickets, or do them one by one."
argument-hint: "[max=N] [dryRun]"
disable-model-invocation: false
---

# Loop Tickets

Work the tickets from this conversation one at a time, running `/implement` on each.

`/to-tickets` produces the pool; this drives it. `/implement` builds each ticket, calling
`/tdd` and `/code-review` itself — do not restate their work here.

## Process

### 1. Capture the pool

The pool is the tickets this session just produced. `/to-tickets` writes them out as it
goes, so take the text from what it wrote — `.scratch/<feature-slug>/issues/*.md`, or the
issues it just created — rather than re-typing it from the conversation, which paraphrases
acceptance criteria. Use the conversation only if it wrote nothing.

Copy them into `~/.claude/loop-tickets/<repo>/tickets.json` — id, title, what to build,
acceptance criteria, blocked-by, `status: "todo"` — before starting the first ticket: a
long run gets compacted, and the tickets are unrecoverable afterwards. Keep the file
outside the repository so the working tree stays clean.

If this session produced no tickets, say so and stop. Never invent a pool.

### 2. Work the frontier, one ticket at a time

Take the lowest-numbered ticket whose blockers are all `done`. For each:

- Branch from the freshly pulled base.
- **Create new test data for this ticket**, tagged with the ticket id so it is
  identifiably yours. Never reset the database, and never reuse or delete an earlier
  ticket's data — reading what is already there is fine and often the point.
- Run `/implement` in a fresh context — a subagent — so ticket N+1 does not inherit
  ticket N's.
- Open the PR and merge it when green.
- Set that ticket's `status` in `tickets.json`, pull the base, and take the next.
  Without the status write the loop re-picks the same ticket forever.

Honour `max=N` if given. Under `dryRun`, open each PR and stop without merging.

### 3. Stop

Stop when the pool is drained, or when a ticket needs a product decision the tickets never
ruled on. Difficulty is not such a decision — a hard ticket, a failing test, or something
inferable from existing precedent all get worked, not skipped.

To skip one: comment on it saying what decision is needed and what it blocks, leave it
open, and move on. Three consecutive skips, or five in total, and stop and report — at
that point the specification is probably incomplete.

## Rules

- One ticket, one branch, one PR. Never two tickets in one PR.
- Each ticket's test data is its own; the database is never reset between tickets.
- Keep pushes on feature branches.
- Ask when a decision is the user's to make. Never guess on one.

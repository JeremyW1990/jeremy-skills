---
name: loop-tickets
description: >-
  Drive a pool of tickets to completion, one at a time, running /implement on each in
  its own fresh context. The missing link between /to-tickets and /implement: one
  ticket, one branch, one PR, one merge, each with newly created test data, on a
  database that is never reset between tickets.
argument-hint: "[max=N] [dryRun] [only=ID,ID] — omit to work the whole pool"
disable-model-invocation: true
---

# Loop the ticket pool

`/to-tickets` produces a pool of tracer-bullet tickets with their blocking edges. This
drives them: pick the next ready ticket, branch, give it fresh test data, hand it to
`/implement`, merge, repeat.

**Composes rather than reimplements.** `/implement` builds the ticket, calling `/tdd` and
`/code-review` itself. This skill owns only what `/implement` has no place for — the
things that go wrong *between* tickets rather than inside one.

**Requires** `/implement` on the current machine. Confirm it resolves before the first
ticket; if it does not, say so and stop rather than improvising a substitute.

**If a compaction or a container replacement has just happened, re-read this file and
`<stateDir>/state.json` before doing anything else.** Every loop-scoped decision lives on
disk precisely because context does not.

**Ask when it matters.** A product decision the tickets never ruled on belongs to the
human. Difficulty never is.

## Inputs

`/loop-tickets [max=N] [dryRun] [only=ID,ID]` — `max=N` caps tickets this invocation,
`dryRun` opens each PR and stops without merging, `only=` restricts the pool.

---

## Step 0 · Materialize the pool

The pool is **the tickets this conversation produced** — never a tracker search, which
returns whatever else is open and cannot tell a ticket just drafted from an issue that has
sat there for months. Capture it to disk before the first ticket, because context is
compacted on long runs.

```sh
slug="$(basename "$(git rev-parse --show-toplevel)")-$(git rev-list --max-parents=0 HEAD | tail -1 | cut -c1-7)"
mkdir -p ~/.claude/loop-tickets/"$slug"/{fixtures,reports}
```

`tail -1` because a repository may have several root commits. **Loop state never goes into
the repository** — a dirty tree breaks the branch and merge steps.

`/to-tickets` publishes what it drafts, so read its artifacts rather than re-typing prose:
`.scratch/<feature-slug>/issues/*.md` in numeric order, or `gh issue view <N>` for exactly
the issues this conversation created. The conversation still decides *which* tickets are in
the pool.

Write `tickets.json` — the authoritative store, an ordered array of
`{id, title, whatToBuild, acceptance[], blockedBy[], issueRef|null, status:"todo"}` — and
`state.json` — `{baseBranch, hasRemote, skipsConsecutive:0, skipsCumulative:0,
heldForHuman:[]}`. Echo the list back with its blocking edges.

**Completion criterion:** `tickets.json` exists, one entry per ticket, each with a
non-empty `acceptance[]`. No tickets in the conversation → resume an existing
`tickets.json`, or stop. Never invent a pool.

## Step 1 · Preflight, before every ticket

Residue impersonates product defects: a leftover scratch table, a disabled trigger or a
torn row turns unrelated suites red while the failure text names none of them. Run the
project's own preflight if it has one, then confirm `git status --porcelain` is empty and —
when there is a remote — that the base branch is current.

Record the base branch's failing test **names** once per base commit, so a red can be
attributed. Re-capture after every merge: the loop itself moves the base.

**Completion criterion:** clean tree, current base, a recorded baseline for this base sha.

---

## The cycle

**1 · Select.** The lowest-ordered ticket whose `status` is `todo` and every ticket in
whose `blockedBy` is `completed`. Re-derive every iteration — an earlier ticket may have
unblocked several. Then confirm it is not already done by reading the code and the tests,
never a design doc's silence; if its premise no longer reproduces, mark it `obsolete` and
select again.
*Criterion:* one ticket id echoed with its blockers shown complete, or a stop with a reason.

**2 · Branch.** `git switch -c "<prefix><ticket-id>"` from the refreshed base. One ticket,
one branch, one PR — never two tickets in one PR, because interleaved failure classes cost
more to untangle than the batching saves.
*Criterion:* HEAD is that branch and the tree is clean.

**3 · Mint the test-data namespace.** `ns="<ticket-id>-$(date +%s)-$(openssl rand -hex 4)"`,
recorded in `<stateDir>/fixtures/<ticket-id>.json` **before any data is created**. The
nonce matters for concurrency: sibling agent lanes share one database, so a bare ticket-id
prefix could collide with another lane's in-flight fixtures. Protocol:
[references/test-data.md](references/test-data.md).
*Criterion:* the manifest exists before the first row is written.

**4 · Implement in a fresh context.** Dispatch one subagent per ticket — it must be able to
edit files, run commands, commit and use `gh`. Give it: the ticket verbatim, its namespace
and the test-data protocol, the baseline failing-test names, and **which GitHub transport
it has** (subagents routinely hold `gh` but not the MCP tools, and a process that halts on
"GitHub tools absent" would otherwise abort every ticket on contact).

Tell it to **run `/implement`** on that ticket, and that it **cannot ask the human** — where
it would pause, it returns instead. Under `dryRun` it opens the PR and stops.

It writes its report to `<stateDir>/reports/<ticket-id>.json` before finishing:
`{status: completed|blocked|skipped|needs-human-merge, pr, issueClosed, notes}`.
*Criterion:* a report parsed into that shape. Nothing parseable → `aborted`.

**5 · Close.** Recover the tree first — the subagent shared it, so commit anything left on
an `aborted` report rather than discarding work. Verify `completed` independently: the PR
is merged, and where an `issueRef` exists it reads `CLOSED`. Raise `needs-human-merge` with
the human. **Then write `status` back to `tickets.json`** — without that write the Select
predicate stays true and the loop re-picks the same ticket forever.

Then: pull the base → re-capture the baseline → preflight → re-derive the frontier. A red
on the base that was not in the pre-merge baseline was introduced by the ticket just merged.
*Criterion:* the ticket's `status` is no longer `todo`, tree clean, HEAD on the base branch.

---

## Skipping, and stopping

**Skip** rather than halting the queue when a ticket meets the pause bar — a product
decision the approved material genuinely never ruled on. A hard implementation, a failing
test, needing more investigation, or an ambiguity inferable from precedent are none of
them; prefer inferring and recording the inference. Comment on the ticket saying which
decision was never ruled on, what answer unblocks it, and what it blocks. Leave it open.

**Skip counts stop the program**: three consecutive, or five cumulative. They live in
`state.json`, never in context — compaction silently zeroes an in-context counter.

**Stop and ask only when:** the pool is drained · an unratified choice would materially
change product behaviour · new credentials or external coordination are needed · the action
would clearly widen scope · the same external blocker recurred with no safe alternative ·
a required system is persistently unavailable · a skip count tripped.

A failing test, a hard ticket, or a real defect in the current ticket are **not** reasons
to stop.

## Guardrails

- **The database is never reset between tickets.** Each ticket adds its own namespaced
  data and reads the accumulated corpus freely; an empty database measures nothing. A
  from-zero rebuild is for a schema-touching ticket only, on a scratch database.
- **A ticket writes only its own rows.** Reading what earlier tickets left is the point.
- **Sweep by class, never by prefix** — a bulk prefix sweep is a database reset in disguise.
- **Classify a command by reading its source, never by running it, not even `--help`.** A
  script with no argument parser ignores the flag and executes; a reset wrapper will drop
  your schemas.
- Keep pushes on feature branches. Never force-push, never bypass commit hooks.
- **Keep closing keywords away from issue references in a PR body**, including when
  describing the trap — the linker ignores negation and quotation alike.
- **One writer at a time over the working tree.** Let each ticket finish before the next.
- **Report a red only beside the baseline at the same scope**, comparing failing test
  *names* — a count can match while one failure was fixed and another introduced.

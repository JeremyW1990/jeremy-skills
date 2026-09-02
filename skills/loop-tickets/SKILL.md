---
name: loop-tickets
description: >-
  Work the tickets in this conversation to completion, one at a time, applying the
  full implementation process to each in its own fresh context. One ticket, one
  branch, one PR, one merge — each with newly created test data, on a database that
  is never reset between tickets.
argument-hint: "[max=N] [dryRun] [only=ID,ID] — omit to work the whole pool"
disable-model-invocation: true
---

# Loop the ticket pool

Take the tickets **this conversation produced** and finish them one at a time. Each
ticket gets a fresh context, its own branch and PR, and its own newly created test data.

**The implementation process ships inside this skill**, at
[references/implement.md](references/implement.md) — the whole process, not a pointer at
any project's own `implement` skill. A project that has a good one is lucky; most resolve
that name to a nine-line stub. Carrying the text is what makes this loop behave the same
in every repository. The test-data protocol is
[references/test-data.md](references/test-data.md).

**If a compaction or a container replacement has just happened to you, re-read this file
and `<stateDir>/state.json` before doing anything else.** Every loop-scoped decision lives
on disk precisely because context does not survive.

**Ask when it matters.** A product decision the tickets never ruled on belongs to the
human — ask via AskUserQuestion. Difficulty is never a reason to ask.

**Arguments.** `max=N` caps tickets this invocation (default: the whole pool). `dryRun`
opens each PR and stops without merging. `only=ID,ID` restricts the pool to those tickets.

---

## Step 0 · Materialize the pool

The pool is **the tickets this conversation produced** — never a tracker search. A search
cannot find it: an "open and unblocked" tracker query returns whatever else is open —
bug reports, debt notes, parent specs — and cannot tell a ticket this conversation drafted
from an issue that has been sitting there for months.

Context is compacted on long runs, so capture the pool to disk first.

1. **State directory.** `~/.claude/loop-tickets/<slug>/`, where `<slug>` is
   `<repo-directory-name>-<first-root-commit-sha7>`:
   ```sh
   slug="$(basename "$(git rev-parse --show-toplevel)")-$(git rev-list --max-parents=0 HEAD | tail -1 | cut -c1-7)"
   mkdir -p ~/.claude/loop-tickets/"$slug"/{fixtures,reports}
   ```
   `tail -1` because a repository may have several root commits; the last is stable.
   Outside a git repository, use the directory name alone.
   **Loop state never goes into the repository** — the working tree must stay clean, and a
   dirty tree breaks the branch and merge steps.

2. **Prefer the artifact over prose.** `/to-tickets` always *publishes* what it drafts, so
   the verbatim text usually exists on disk already. For the tickets this conversation
   produced — and only those — read them rather than re-typing:
   - local mode → `.scratch/<feature-slug>/issues/*.md` in numeric order
   - tracker mode → `gh issue view <N> --json number,title,body` for exactly the issue
     numbers this conversation created, recording each as `issueRef`
   - neither → take the text verbatim from the conversation
   This is not a tracker *search*; the conversation still decides **which** tickets are in
   the pool.

3. **Write `tickets.json`**: an ordered array of
   `{id, title, whatToBuild, acceptance[], blockedBy[], issueRef|null, status:"todo"}`.
   This file is the **authoritative** ticket store: `status` is read by Select and written
   by Close.

4. **Write `state.json`**:
   ```json
   {"createdAt":"…","stateDir":"…","baseBranch":"…","hasRemote":true,
    "baseline":{"sha":null,"scope":null,"command":null,"failing":[]},
    "skipsConsecutive":0,"skipsCumulative":0,"heldForHuman":[],"bindings":{}}
   ```

5. Echo the ticket list back with its blocking edges, and say where the state directory is.

**Completion criterion:** `tickets.json` exists, its entry count equals the number of
tickets this conversation produced, and every entry has a non-empty `acceptance[]`.
If the conversation holds no tickets, resume from an existing `tickets.json`; if there is
none, say so and stop — never invent a pool.

## Step 1 · Bind the project, once

Fill `state.json.bindings`, confirming with the human before the first ticket:
`baseBranch`, `branchPrefix`, `bootstrap`, `lint`, `typecheck`, `build`, `test`,
`testScopeForBaseline`, `seed`, `resetDb`, `preflight`, `github` (`mcp` | `gh` | `none`),
and `readFirst` (the project's own agent guide and architecture docs).

Detect from package scripts, Makefile, justfile and CI config. **Classify a candidate
command by reading its source, never by running it** — see the warning in
[references/test-data.md](references/test-data.md).

Record `hasRemote` from `git remote` being non-empty.

**Completion criterion:** every binding holds a value or the explicit string `absent`.

## Step 2 · Preflight, before every ticket

Residue is the highest-yield check and it impersonates product defects: a leftover scratch
table, a disabled trigger or a torn row turns unrelated suites red while the failure text
names none of them.

Run `bindings.preflight` if present. Then always: `git status --porcelain` is empty; and
when `hasRemote`, `git fetch --prune` and confirm the base is not behind its remote.

**Baseline.** Capture once per base-branch commit, on the base branch, before any ticket
branch exists: run `bindings.test` at `bindings.testScopeForBaseline` and record
`{sha, scope, command, failing[]}` — the failing test **names**, not a count. Re-capture
after every merge, because the loop itself moves the base.

**Completion criterion:** the tree is clean, the base is current (or `hasRemote` is false),
and `state.json.baseline.sha` equals the current base commit.

---

## The cycle · one ticket at a time

### 1 · Select
The lowest-ordered ticket in `tickets.json` whose `status` is `todo` and every ticket named
in whose `blockedBy` has `status: completed`. Honour `only=`. Re-derive at the start of
**every** iteration — an earlier ticket may have unblocked several, and a human may have
changed something.

Then check it is not already done: read the code and the tests, never a design doc's
silence. Absence of a marker is evidence of nothing. If its premise no longer reproduces,
set `status: obsolete` with a note and select again.

**Completion criterion:** one ticket id is echoed with each of its `blockedBy` shown as
`completed`, or the loop stops with "no ready ticket" and the reason.

### 2 · Branch
`git switch -c "<branchPrefix><ticket-id-lower>"` from the refreshed base. One ticket, one
branch, one PR — never two tickets in one PR. The reason is attribution, not policy:
interleaved failure classes cost more to untangle than the batching saves.

**Completion criterion:** `git rev-parse --abbrev-ref HEAD` is that branch and
`git status --porcelain` is empty.

### 3 · Mint the test-data namespace
```sh
ns="<ticket-id-lower>-$(date +%s)-$(openssl rand -hex 4)"
```
Write `<stateDir>/fixtures/<ticket-id>.json` **before any data is created**. The nonce is
load-bearing: sibling agent lanes share the database, so a bare ticket-id prefix could
collide with another lane's in-flight fixtures. Protocol:
[references/test-data.md](references/test-data.md).

**Completion criterion:** that manifest file exists on disk before the first row is written.

### 4 · Dispatch a fresh subagent
Use the **Task tool** with `subagent_type: general-purpose` — it must be able to edit
files, run commands, commit, and use `gh`. One subagent per ticket, so each starts clean.

Put in the prompt:

- the ticket verbatim from `tickets.json`, including `issueRef` (or that there is none)
- **read `~/.claude/skills/loop-tickets/references/implement.md` in full before doing
  anything, and follow it as the authoritative process**
- the bindings, the base branch, and the baseline `{scope, command, failing[]}`
- its namespace, and to follow `~/.claude/skills/loop-tickets/references/test-data.md`
- **the GitHub transport**: name whether `mcp__github__*` or `gh` is available. Subagents
  routinely lack the MCP tools while `gh` is authenticated, and a process step that halts
  on "GitHub tools absent" would otherwise abort every ticket on contact.
- **that it cannot ask the human** — it has no AskUserQuestion tool. Where the process
  says to pause, it returns instead.
- `dryRun` if set: open the PR ready, do not merge.
- **the return contract**: write the report to
  `<stateDir>/reports/<ticket-id>.json` *before finishing*, and also emit it as the last
  line of the final message prefixed `LOOP-TICKETS-REPORT: `. Shape:
  ```json
  {"status":"completed|blocked|skipped|needs-human-merge","pr":null,"issueClosed":false,
   "p0":[],"p1":[],"p2":[],"baselineDeltas":[],"fixturesCreated":[],"notes":""}
  ```

**Completion criterion:** a report parsed into that shape, from the file or the final line.
A subagent that returns nothing parseable is recorded `{"status":"aborted"}`.

### 5 · Close the iteration

**First, recover the tree** — the subagent shared it. If `git status --porcelain` is
non-empty on an `aborted` report, commit it to the ticket branch (never discard work) and
record the branch in the ticket's note.

Then, by status:

| status | action |
|---|---|
| `completed` | verify independently: the PR is merged, and where `issueRef` is set it reads `CLOSED` |
| `needs-human-merge` | ask the human via AskUserQuestion; merge, or append to `heldForHuman` |
| `skipped` | increment both skip counters |
| `blocked` / `aborted` | record the reason; treat as skipped for counting |

**Write back to `tickets.json`**: set that ticket's `status` to `completed`, `skipped`,
`blocked` or `held`. Nothing else advances the loop — without this write the Select
predicate stays true and the loop re-picks the same ticket forever.

Reset `skipsConsecutive` to 0 on a `completed`; increment on anything else.

Then run the between-tickets sequence:
**pull the base → re-capture the baseline (the base moved) → preflight → re-derive the
frontier.** A red on the base that was not in the pre-merge baseline was introduced by the
ticket just merged; fix it before starting the next.

Decrement the `max=N` budget and stop when it reaches zero.

**Completion criterion:** the ticket's `status` in `tickets.json` is no longer `todo`, the
tree is clean, and HEAD is the refreshed base branch.

---

## Skipping, and stopping

**Skip** rather than halting the queue when a ticket meets the pause bar. Comment on it
saying three things: which decision was never ruled on, what answer unblocks it, and what
it blocks. Leave it open, do not relabel it, and take the next ticket.

The skip bar is **exactly** the pause bar. A hard implementation, a failing test, needing
more investigation, or an ambiguity inferable from existing precedent are none of them —
prefer inferring from precedent and recording the inference.

**Skip counts stop the program**: three consecutive, or five cumulative. They live in
`state.json`, never in context — compaction silently zeroes an in-context counter and
disables this guard.

**Stop and ask only when:** the pool is drained · an unratified choice would materially
change product behaviour · new credentials or external coordination are needed · the action
would clearly widen scope · the same external blocker recurred with no safe alternative ·
the tracker, the database or another required system is persistently unavailable · a skip
count tripped.

A failing test, a hard ticket, needing more investigation, or a real P0/P1 in the current
ticket are **not** reasons to stop.

## Hard guardrails

- **The database is shared and is never reset between tickets.** Each ticket *adds* its own
  namespaced data; the accumulated corpus is the instrument that finds defects, and an
  empty database measures nothing. A from-zero rebuild is for a schema-touching ticket
  only, on a uniquely-named scratch database.
- **A ticket reads the whole corpus and writes only its own rows.** Reading what earlier
  tickets left is the point; mutating or deleting it is not.
- **Sweep by class, never by prefix.** A bulk prefix sweep is a database reset in disguise:
  the fixtures it removes are often exactly what a non-vacuity guard asserts is present.
- Keep pushes on feature branches; integration happens through the PR. Never force-push and
  never bypass commit hooks.
- **Keep closing keywords away from issue references in a PR body** — including when
  describing the trap. The linker ignores negation and quotation alike, and has fired twice
  on the same issue, the second time inside the PR that documented the first.
- **One writer at a time over the working tree.** The subagent shares it, so let each
  ticket finish before the next begins.
- **Report a red only beside the baseline at the same scope**, comparing sorted failing
  test *names* — a count can match while one failure was fixed and another introduced.
- Loop state lives in `<stateDir>`, never in the repository.

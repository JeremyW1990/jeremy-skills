---
name: loop-tickets
description: "Drain a dependency-ordered implementation ticket pool, one branch and PR per ticket, with resumable state, project learning notes, and non-duplicative validation. Use after to-tickets or when asked to work a ticket queue end to end."
---

# Loop Tickets

Work one implementation ticket at a time. The orchestrator owns queue state, reusable
project context, review, final validation and PR delivery; the selected runner owns the
implementation method and focused development feedback. Use `$loop-tickets` in Codex
and `/loop-tickets` in Claude Code.

User choices set scope and options; exact tickets set acceptance criteria and blockers;
current project instructions constrain workflow; actual code/configuration reveals
behavior and coverage. Project notes are advisory navigation only.

## Inputs

`loop-tickets [runner=<skill>] [validation=project|local] [review=project|none|code-review] [max=N] [noMerge|dryRun]`

Omitted options follow the project. The runner defaults to `implement` only when neither
the user nor project selects one. Resolve `review=project` to a concrete project mechanism;
if none exists, inherited review becomes `none`, while an explicit user request must be
reported. `review=none` removes a separate review pass, not acceptance checks.
`validation=local` governs checks started by this skill; required CI triggered by a push
or PR must still pass. `dryRun` is a legacy alias for `noMerge`: both may push and open a
PR but never merge. No option authorizes deployment.

## 1. Capture the pool and state

Read the exact published issues or ticket package selected by the user. Otherwise use
what `to-tickets` wrote this session: `.scratch/<feature-slug>/issues/*.md` or its
published issues. Never reconstruct acceptance criteria from a conversation summary.
If no pool can be located, report the paths and trackers searched and stop.

Use one stable `<repo-key>` for a canonical remote repository across its worktrees; if
there is no remote, derive it from the canonical repository root plus a short hash. Store
state outside the working repository:

- Codex: `${CODEX_HOME:-$HOME/.codex}/loop-tickets/<repo-key>/`
- Claude Code: `~/.claude/loop-tickets/<repo-key>/`

Keep each pool in its own child directory and `PROJECT-NOTES.md` beside those directories.
Reuse matching state. Normalize legacy state before trusting it, preserving ticket status
and evidence rather than resetting them.

Before implementation:

- Persist each ticket's exact source/version, title, acceptance criteria, blockers and
  status. Validate dependency closure and cycles once against the authoritative tracker.
- Use one canonical ticket status: `todo`, `working`, `pr_open`, `validating`,
  `validated`, `blocked` or `done`. Derive active ticket, counts and frontier from ticket
  statuses on every read; stored summaries are caches and never override them.
- Record runner, base, merge command, review policy, validation location and PR/CI
  triggers. Batch-fetch the pool; on resume refresh only relevant changed sources.
- Persist only redacted identifiers, fingerprints and evidence pointers. Never record
  secrets, credential-bearing URLs, environment values, tokens, customer data or fixture
  payloads in state, receipts or notes.

## 2. Prepare reusable project context

Read current user/ticket and project instructions, then use `PROJECT-NOTES.md` to locate
likely validation entrypoints and known costs. Verify every reused claim against its
referenced files. Inspect only the current pool's packages and reachable validation
configuration, expanding discovery when coverage is missing or an input changed.

Build a lazy project validation profile covering relevant commands, selectors,
prerequisites, dependents, wrapper/subsumption edges, triggers, stateful resources and
observed costs. Fingerprint validation topology (instructions, task graph, scripts, hooks,
CI) separately from dependency/runtime state (manifests, lockfiles, toolchain, services,
artifacts), invalidating only affected facts. Reuse this profile across pools while valid,
but derive ticket-specific acceptance coverage for every new pool. A profile is not proof
that a current candidate passed.

Prepare dependencies, generated artifacts and services only when needed. Reuse healthy
services and sound task caches. Do not force caches, clean builds, reinstall dependencies
or restart services unless policy requires it or a defect is being diagnosed.

Keep the full profile in durable state. Give each ticket worker only the relevant
projection: current policy, the complete ticket, exact source pointers/hashes, reachable
validation facts, still-valid receipts and verified note excerpts.

## 3. Maintain the project note

Before creating or updating the note, read
[references/project-notes.md](references/project-notes.md) and follow its schema,
freshness, locking and size rules. Refresh it at start/resume, after a ticket completes or
becomes active or blocked, before a long external wait, after completion, and when the
invocation stops—not after every command or short-lived transition.

Derive the note from pool state and receipts. Never create a test, worker, branch, commit,
push or PR for the note itself. Note-writing failure must not alter queue state; regenerate
it at the next boundary. Its future-ticket lessons do not change the approved pool or
automatically become project policy.

## 4. Work the frontier

Resume an active ticket first. Otherwise choose an eligible `todo` ticket whose blockers
are satisfied. Follow authoritative priority; among equals prefer critical-path impact
and a reusable healthy environment, with ticket number as the final tie-breaker. Exclude
`blocked` tickets until their reason is resolved.

1. **Synchronize at ticket boundaries.** Fetch/fast-forward the base before the first
   ticket and after each merge. Branch from that base; the worker does not pull again.
   Keep one ticket per branch/PR unless authoritative instructions combine delivery.
2. **Create only necessary fixtures.** Use run/ticket-owned mutable data when isolation is
   needed and reuse factories. Do not reset shared data or prepare a database for work
   that does not need one. Rebuild only a run-owned scratch environment under its contract.
3. **Use one fresh worker per ticket.** In Codex use `spawn_agent` with
   `fork_turns: "none"`; otherwise use the host equivalent. Reuse that worker for all
   repairs and validation feedback on the ticket. The orchestrator alone changes queue
   state and merges unless project instructions delegate them.
4. **Develop with focused feedback.** Let the runner choose its development method. It
   runs focused and uncovered ticket-specific checks, inspects its diff and prepares a
   stable candidate. Tell it not to run the broad final gate, separate review or
   CI-triggering push unless the orchestrator explicitly assigns that responsibility.
5. **Persist meaningful transitions.** Atomically record status, branch/head, PR URL,
   validation plan/receipt and blocker at `working`, `pr_open`, `validating`, `validated`,
   `blocked` and `done`. `done` requires a confirmed merge. Inspect uncertain external
   results before retrying; a closed unmerged PR is not done.
6. **Review before the costly final gate.** Assign exactly one review owner. The
   orchestrator owns it by default and consumes a runner-owned review only when project
   instructions delegate it. Run one planned broad review; after review-driven edits,
   verify resolved findings and inspect the changed final diff without repeating
   unchanged review work. `review=none` creates no reviewer/evaluator.
7. **Validate and deliver one successful evidence set.** Assign each required assurance
   a primary evidence source: focused work, hook, local gate or CI. Run all mandatory
   gates even when they overlap, but add no manual duplicate. Detect triggers before
   choosing local validation before the first push versus CI as final owner. Push/open
   only a stable candidate unless the required validator is remote-only.
8. **Finalize, then advance.** Confirm the external merge first. Atomically update the
   authoritative queue: save the merge receipt, mark `done`, clear the active ticket and
   derive counts/frontier from ticket statuses. Regenerate the advisory note separately,
   synchronize the base once and do not rerun a full suite on the same post-merge tree.

## 5. Validate without duplication

- Prefer one affected dependency graph over root commands that repeat builds or tests.
  Retain required stateful, migration, concurrency and end-to-end checks it does not cover.
- Each receipt records expanded command/selector, purpose, outcome, duration, tested
  head/tree and base, relevant input/runtime fingerprints, cache use and artifact pointers.
  Stateful checks add redacted build, schema, fixture, service and runtime fingerprints.
  Point to existing logs; do not copy logs or artifacts into queue state. Atomically update
  a compact per-pool metrics index (runs, duration, failures, reuse and failure signatures)
  so note generation never rescans the full receipt history.
- Record an evidence-retention policy during preparation. Deduplicate artifacts and keep a
  compact attempt index. On pool completion, follow project policy to compress/archive or
  remove only run-owned superseded artifacts; retain required audit/resume evidence and
  never let cleanup become its own validation run.
- Reuse a pass by default only for the exact candidate with matching inputs. Cross-commit
  reuse requires a trustworthy affected-graph or cache proof. After changes, invalidate
  only affected evidence; rerun a lightweight controller as needed, but rerun a full
  subcommand only when its input closure or project policy requires it.
- Order selected checks for fast failure and run expensive gates only on a stable
  candidate. A failed final attempt is not a reason to replay already-valid checks.
  Inspect unreached checks for the same broken invariant before the next costly attempt.
- If a failure may be intermittent, obey any stricter project policy; otherwise retry the
  failing case/job at most once per candidate and normalized signature. A changed outcome
  is only `suspected-intermittent`. Then diagnose or record a genuine infrastructure
  blocker; never weaken assertions merely to obtain green.
- Parallelize only resource-independent checks. Keep shared databases and memory-heavy
  work coordinated. Never report a skipped command as freshly passed.

## 6. Stop

Honor `max=N` as tickets completed in this invocation. Under `noMerge`/`dryRun`, validate
and open the current PR, then stop without merging or unlocking dependents.

Continue while any eligible ticket remains. Stop when the pool is drained or every
remaining ticket is blocked on a genuine product decision or external dependency.
Difficulty, failure or an inferable implementation choice is work to resolve, not skip.
Persist blockers and affected dependents; comment on the tracker only when authorized.
Refresh `PROJECT-NOTES.md`, then report the queue result, note path and most useful
verified learnings.

Keep feature pushes, required checks and the user's scope intact throughout the loop.

---
name: loop-tickets
description: "Work a dependency-ordered pool of implementation tickets to completion, one ticket per branch and PR, with persistent state and deduplicated validation. Use after to-tickets or when asked to drain a ticket queue end to end."
---

# Loop Tickets

Work one implementation ticket at a time. This skill owns queue state, workflow
selection, validation coverage and PR delivery; the selected runner owns implementation.
Use the host's skill syntax: `$loop-tickets` in Codex, `/loop-tickets` in Claude Code.

## Inputs

`loop-tickets [runner=<skill>] [validation=project|local] [review=project|none|code-review] [max=N] [dryRun]`

Omitted options follow the active project. Explicit user choices override inherited
runner instructions. `review=none` removes separate Code Review/Evaluator passes, not
acceptance tests. `validation=local` runs checks locally; it neither changes remote
settings nor bypasses required remote checks. No option authorizes deployment.

## 1. Prepare once

Read the exact published issues or ticket package the user selected. Otherwise use
what `to-tickets` wrote this session: `.scratch/<feature-slug>/issues/*.md` or its
published issues. Never reconstruct acceptance criteria from conversation summaries.
If none can be located, report the paths searched and stop; do not invent a pool.

Persist each ticket's exact source, source hash/version, title, acceptance criteria,
blockers and status before implementation. In Codex use
`${CODEX_HOME:-$HOME/.codex}/loop-tickets/<repo>/<pool>/`; in Claude Code use
`~/.claude/loop-tickets/<repo>/<pool>/`. Reuse an existing matching state file,
including an older layout, rather than resetting its statuses. Keep unrelated pools
separate and all queue files outside the working repository.

- Validate dependency closure and cycles once against the authoritative tracker.
  A blocker must be in the pool, confirmed satisfied externally, or explicitly
  recorded as an unresolved external dependency with its source and reason.
- Record runner (default `implement`), base branch, merge command, review policy and
  validation location. Fetch the selected pool in a batch; do not reload every issue
  before every ticket. Refresh relevant source/dependency changes and recheck on resume.
- Inspect actual scripts, hooks, CI and applicable project instructions. Record a
  compact coverage map: underlying commands, test-file selections, dependency tasks
  and required database/browser checks. Expand wrappers; a command name proves no coverage.
- Prepare a shared context packet with current policy, exact source pointers and file
  hashes. Pass the current ticket in full plus relevant references to each worker;
  do not make every worker rediscover the entire backlog or retired project workflows.
  Refresh changed instructions and read relevant source sections; the packet is not
  permission to ignore current project rules or replace the exact ticket.
- Prepare dependencies and local services only when needed. Reuse a healthy stack;
  install when dependencies/runtime changed or are missing, and generate when inputs
  changed or outputs are missing. Record setup's actual build/generation coverage.

Reuse this preparation while its inputs remain unchanged. Never reuse an old pool's
test map merely because it belongs to the same repository.

## 2. Work the frontier

Resume an active ticket first. Otherwise take the lowest-numbered `todo` ticket whose
blockers are satisfied. Exclude tickets marked `blocked` until their reason is resolved.

1. **Synchronize once at the ticket boundary.** The orchestrator fetches/fast-forwards
   the base before the first ticket and after each merge. Branch the next ticket from
   that synchronized base; the worker does not repeat the pull. Keep one ticket per
   branch/PR unless authoritative instructions explicitly combine their delivery.
2. **Prepare only necessary fixtures.** For persistent test data, use ticket/run-tagged
   clients and GL inputs. Reuse fixture factories, not another ticket's mutable client.
   Pure or documentation work needs no artificial database setup. Do not reset the
   existing database or remove earlier tickets' data. An explicitly authorized,
   run-owned scratch database may be rebuilt/cleaned according to its test contract.
3. **Run the selected runner in a fresh worker context.** In Codex use `spawn_agent`
   with `fork_turns: "none"`; otherwise use the host's equivalent subagent. Pass the
   exact ticket, workflow policy, context packet, coverage map and current evidence.
   The worker can edit, test, commit and push its feature branch. Only the orchestrator
   owns queue transitions and final merge unless the project explicitly delegates them.
4. **Use focused TDD during development.** Accepted ticket/contract seams are already
   agreed; do not ask again. Under `review=none`, do not invoke Code Review or an
   Evaluator. The worker inspects its own diff and satisfies each acceptance criterion.
5. **Persist progress at meaningful boundaries.** Atomically save phase, branch/head,
   PR URL, validation plan/receipt and any blocker. Distinguish `working`, `pr_open`,
   `validating`, `validated`, `merged` and `blocked`; retain completed tickets as `done`.
   On interruption or an uncertain push/merge result, inspect the existing branch/PR
   before recreating it or repeating work. A closed unmerged PR is not completion.
6. **Open the PR, then perform final validation once.** Use the project's existing
   validation-and-merge command when present; do not pre-run its suite for a handoff.
   Otherwise run the selected checks and merge when green. Bind the plan/results to
   the tested head/base and keep the candidate clean. Local validation mode dispatches
   no duplicate CI and waits for no nonexistent status check.
7. **Finish and advance.** Confirm the merge, record `done` and the merge receipt,
   synchronize the base once, then take the next ticket. Do not run a post-merge
   baseline/full suite on the same resulting tree.

## 3. Validate without duplication

- Run one affected dependency graph instead of separate root lint/build/typecheck/test
  graphs that repeat builds. If a build really embeds a typecheck, count it once;
  retain separate checks for packages whose build only bundles. Select dependents too.
- Deduplicate tests by actual file/case coverage, not suite names. A full suite plus
  overlapping named suites is not additional evidence. Keep required unselected
  integration, migration, RLS, concurrency and browser checks. Do not substitute a
  fabricated engine result for a real detection/upload acceptance flow.
- Focused RED/GREEN runs provide development feedback. Do not repeatedly run the whole
  final gate after each edit, commit or push. A hook that actually runs a required check
  may supply its evidence; an identity-only hook supplies none.
- On failure, fix and rerun affected checks. Reuse a passed result only when its recorded
  inputs, tool/runtime configuration, needed artifacts and relevant environment remain
  valid. Database/browser results need more than an unchanged source hash. Never label
  a skipped command as freshly passed. A receipt alone is not a cache; inspect what the
  runner actually supports before promising resume or cross-commit reuse.
- Reuse existing sound task caches; do not enable caching with undeclared outputs or
  environment inputs. Parallelize only checks whose resources are independent; shared
  database/reset operations remain coordinated. Builds and memory-heavy checks must fit
  the machine. No blanket cache or concurrency increase is required for a solo developer.
- Replan if the candidate or base changes. A single-developer queue normally avoids that
  work through serial delivery, but still verifies that it is merging the tested code.

## 4. Stop

Honor `max=N` as the number of tickets completed in this invocation. Under `dryRun`,
build and validate the current ticket, open its PR, then stop without merging. Do not
unlock dependent tickets from unmerged work. Use a validation-only command or run
selected checks separately if the project merge command always merges.

Stop when the pool is drained or its remaining tickets are blocked on decisions or
external dependencies. Difficulty, a failing test, or an inferable implementation choice
is work to resolve, not a reason to skip a ticket.

If a genuine unspecified product decision blocks a ticket, save `blocked` with the
reason and affected dependencies, report it and continue other eligible tickets.
Comment on the issue only when that communication is authorized. Stop and report after
three consecutive such skips or five total. Do not repeatedly select the same blocker.

Keep feature pushes, required checks and the user's scope intact throughout the loop.

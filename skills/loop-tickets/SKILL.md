---
name: loop-tickets
description: "Drain a dependency-ordered implementation ticket pool, one branch and PR per ticket, with resumable state, repository-local project notes, project-authorized focused testing, and optional code review. Use after to-tickets or when asked to work a ticket queue end to end."
---

# Loop Tickets

Work one implementation ticket at a time. The orchestrator owns queue state, ticket
selection, optional review, PR delivery and merge; the ticket worker owns implementation
and focused feedback. Use `$loop-tickets` in Codex and `/loop-tickets` in Claude Code.

User choices set scope; exact tickets set acceptance criteria and blockers; project
instructions constrain workflow; actual code and test configuration reveal behavior.
Project notes are advisory only.

## Input

`loop-tickets [review=on|off]`

Review defaults to `on`. `review=on` runs exactly one separate code-review pass per ticket
on its stable candidate; `review=off` omits that pass. Neither setting changes project
test authorization, repository-managed checks or mandatory repository protections.
Resolve the implementation runner from project instructions, otherwise use `implement`.
Honor other user constraints expressed in natural language without inventing more flags.
No option authorizes deployment.

## 1. Capture the pool and durable state

Read the exact published issues or ticket package selected by the user. Otherwise use
what `to-tickets` wrote this session: `.scratch/<feature-slug>/issues/*.md` or its
published issues. Never reconstruct acceptance criteria from a conversation summary.
If no pool can be located, report the paths and trackers searched and stop.

Use one stable `<repo-key>` for a canonical remote repository across worktrees; if there
is no remote, derive it from the canonical repository root plus a short hash. Keep
authoritative queue state outside the working repository:

- Codex: `${CODEX_HOME:-$HOME/.codex}/loop-tickets/<repo-key>/<pool>/`
- Claude Code: `~/.claude/loop-tickets/<repo-key>/<pool>/`

Persist each ticket's exact source/version, title, acceptance criteria, blockers, branch,
PR and status. Use one status enum: `todo`, `working`, `pr_open`, `validating`, `validated`,
`blocked` or `done`; `done` requires a confirmed merge. Derive the active ticket, counts
and frontier from ticket statuses on every read rather than trusting cached summaries.
Validate dependency closure and cycles once, and batch-fetch the pool.

Persist only redacted identifiers, compact receipts and evidence pointers. Never record
secrets, credential-bearing URLs, environment values, tokens, customer data or fixture
payloads.

## 2. Maintain the tracked project note

The note lives at `<current-ticket-worktree-root>/.loop-tickets/project-notes.md`. It is
tracked with the project and shared through the same ticket PR as the implementation;
authoritative queue state remains outside the repository.

Before creating or updating the note, read
[references/project-notes.md](references/project-notes.md). Read the synchronized base
version at start/resume. After authorized focused tests (or a recorded skip) and optional
review, update it before the ticket's first push, and reconcile it if the target later
advances. Make it an
`effective-on-merge` projection: mark the current ticket `done` and derive the next
frontier, but do not claim a PR URL, merge SHA or remote CI result that does not exist
yet. If the PR does not merge, the projection never reaches the target branch; if it
merges, the projected state becomes true.

Stage it with the ticket's final local commit or amend before the first push. Never create
a note-only branch, commit, push, PR, CI run or test. Failure to safely update/stage the
required note blocks ticket delivery unless the user waives it. Future-ticket lessons do
not alter the approved pool or automatically become project policy.

## 3. Prepare once, then only what the ticket needs

Read current project instructions, then use the note to locate likely commands, test
selectors and known waste patterns. Verify reused claims against current files. Inspect
only code paths, packages, hooks and CI triggers relevant to the current ticket; do not
inventory the full repository or reload the backlog for every worker.

Prepare dependencies, generated artifacts, fixtures and services only when the current
ticket needs them. Reuse healthy services, fixture factories and sound caches. Do not
force caches, clean builds, reinstall dependencies or reset shared data unless required
to diagnose a demonstrated defect.

Give the worker the complete ticket plus only relevant project rules, source pointers,
test selectors, current receipts and verified note excerpts.

## 4. Work the frontier

Resume an active ticket first. Otherwise choose an eligible `todo` ticket whose blockers
are satisfied. Follow authoritative priority; use ticket number as the final tie-breaker.

1. **Synchronize before every ticket.** After selecting a ticket but before creating its
   branch/worktree or spawning its worker, fetch the latest `develop` from the repository's
   GitHub remote and fast-forward the local `develop` base to that remote-tracking commit.
   Repeat this immediately before every ticket, including a resumed ticket, even if the
   preceding ticket just merged or an earlier fetch appeared current. If the authoritative
   queue explicitly targets another branch, substitute that exact target for `develop` but
   keep the same mandatory per-ticket refresh. Branch from the freshly synchronized commit;
   the worker does not pull again. The orchestrator refreshes the target again before merging
   because other agents may have advanced it. Keep one ticket per branch/PR unless
   authoritative ticket text combines delivery.
2. **Use one worker per ticket.** In Codex use `spawn_agent` with `fork_turns: "none"`;
   otherwise use the host equivalent. Reuse the same worker for all repairs on that
   ticket. Only the orchestrator changes queue state or merges.
3. **Implement with authorized focused tests.** The worker runs only the smallest
   selectors proving the current ticket when project instructions and the current
   conversation authorize tests; otherwise it records them as unrun and inspects the
   diff to prepare a stable candidate. It does not run a full regression suite, a broad
   final gate, a separate review or an intermediate CI-triggering push.
4. **Review if enabled.** Under `review=on`, the orchestrator invokes the configured
   project reviewer or `code-review` once and supplies existing test receipts or unrun
   status. Tell the reviewer to inspect standards/spec compliance without running tests.
   After review fixes, rerun only directly affected current-ticket selectors when
   authorized. Under `review=off`, do not create a reviewer/evaluator.
5. **Deliver once stable.** After authorized focused tests or a recorded skip and optional
   review, regenerate the effective-on-merge note and stage its exact path with the
   ticket's final local commit.
   Push the first stable candidate and open one PR; avoid intermediate CI-triggering
   pushes. A repair after target movement may require another stable-candidate push.
   Allow each mandatory hook, merge command, branch-protection and CI mechanism to run
   as configured by the repository; do not bypass them or duplicate checks on the same
   inputs. Observe their results as described in section 6. Merge only when required
   checks and protections are satisfied, and current-ticket acceptance has sufficient
   evidence. If a required acceptance criterion cannot be established without an unauthorized test,
   record the blocker rather than treating that test as passed.
6. **Finalize, then advance.** Confirm the merge, atomically mark the ticket `done`, clear
   the active ticket and derive the new frontier in external state. Confirm the projected
   note landed with the PR and select the next ticket. Before that ticket starts, repeat
   the mandatory synchronization in step 1; no earlier fetch counts for it. Do not create
   a follow-up note commit.

## 5. Focused development test policy

Project and current-conversation test authorization takes precedence over this skill.
When project policy requires explicit execution authorization, a ticket asking for test
coverage authorizes writing tests, not executing them.

- When tests are authorized, run only explicit cases/files/selectors that prove the
  current ticket's acceptance criteria or directly changed behavior.
- Do not manually run repository-wide, package-wide or full regression *test* suites.
  Do not run tests from earlier tickets merely because they ran before; include one only
  when the current ticket changes the behavior it exercises.
- Never manually repeat a passed test on unchanged relevant inputs. After a code change,
  rerun only authorized selectors whose behavior or inputs changed. After a failure,
  diagnose and rerun the failed or invalidated selector, not every current-ticket test.
- If no focused selector exists, add one when the ticket calls for test coverage;
  otherwise use the narrowest supported selector if execution is authorized and record
  the limitation. Never fall back to a full suite merely because selection is inconvenient.
- Broader mandatory hooks or CI may run automatically where the project permits them.
  Follow the repository's configured lifecycle and result-reuse rules; do not add a
  duplicate manual check. If project policy requires explicit authorization for tests,
  do not deliberately trigger a hook or workflow that runs them without that authorization.
  Report any conflict instead.
- Record selector, purpose, outcome or `unrun`, duration when applicable, tested head
  and relevant input fingerprints in a compact receipt. Identify acceptance criteria
  still blocked by unrun tests. Never call a skipped or inherited result freshly passed.

## 6. Observe repository checks and merge

The repository owns check commands, scope, timing, hooks and CI/CD configuration. This
skill does not define or configure them, add a separate static gate or manually repeat
repository-managed checks. Follow the repository's documented commit, push and merge
workflow and observe the required local results or remote statuses it produces.

Keep the current ticket active while checks are pending. Record the candidate identity,
required check outcomes and compact evidence pointers in external state. Accept only
results the repository considers valid for the current candidate. Missing, pending,
failed, cancelled or stale results are not success; do not infer success from the absence
of remote checks when the repository requires local evidence.

A failed hook or CI check stays on the current ticket and goes back to the same worker
for diagnosis and repair. Preserve its branch and any existing PR, then let the
repository's normal workflow check the repaired candidate. Rerun invalidated focused tests only when
authorized. Do not advance to another ticket while required checks remain unsatisfied.
If execution is unavailable or blocked externally, persist the blocker on this ticket
and stop the queue until it can resume; never mark the ticket done or skip ahead.

Before merging, fetch the latest target and PR heads and follow the repository's merge
command and protections. Resolve conflicts on the current ticket branch with the same
worker. If the target advanced, reconcile changed project instructions and the tracked
project note; never merge a stale projection. Let the repository determine any required
integration checks and whether prior results still apply. Merge only when its required
checks and protections are satisfied and the ticket's acceptance has sufficient evidence.
Confirm the PR actually merged and any repository-required merge verification completed
before marking the ticket done or starting the next ticket.

## 7. Stop

Continue while any eligible ticket remains, subject to section 6: unsatisfied required
checks keep the current ticket active for repair or waiting; never skip it for another
eligible ticket. Stop when required check execution is externally blocked, the pool is
drained, or every remaining ticket is blocked on a genuine product decision, policy
conflict or external dependency. Difficulty, a test failure or an inferable implementation
choice is work to resolve, not skip. Persist blockers and affected dependents; comment on the tracker only
when authorized. Report the queue result, tracked note path, reusable learnings and any
newer queue state that could not be published because no implementation PR existed.

Keep feature pushes, required protections and the user's scope intact throughout the loop.

---
name: loop-tickets
description: "Drain a dependency-ordered implementation ticket pool, one branch and PR per ticket, with resumable state, repository-local project notes, focused current-ticket tests, and optional code review. Use after to-tickets or when asked to work a ticket queue end to end."
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
on its stable candidate; `review=off` omits that pass. Neither setting removes focused
ticket tests, the merge-time static gate or mandatory repository protections. Resolve the
implementation runner from project instructions, otherwise use `implement`. Honor other
user constraints expressed in natural language without inventing more flags. No option
authorizes deployment.

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
version at start/resume. After focused tests and optional review, update it before the
ticket's first push, and reconcile it if the target later advances. Make it an
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

1. **Synchronize between tickets.** Fetch/fast-forward the target branch before the first
   ticket and after each merge. Branch from that base; the worker does not pull again.
   The orchestrator refreshes the target again at the merge gate because other agents
   may have advanced it. Keep one ticket per branch/PR unless authoritative ticket text
   combines delivery.
2. **Use one worker per ticket.** In Codex use `spawn_agent` with `fork_turns: "none"`;
   otherwise use the host equivalent. Reuse the same worker for all repairs on that
   ticket. Only the orchestrator changes queue state or merges.
3. **Implement with focused tests.** The worker runs only the smallest selectors proving
   the current ticket, inspects its diff and prepares a stable candidate. It does not run
   a full regression suite, a broad final gate, a separate review or an intermediate
   CI-triggering push.
4. **Review if enabled.** Under `review=on`, the orchestrator invokes the configured
   project reviewer or `code-review` once and supplies existing test receipts. Tell the
   reviewer to inspect standards/spec compliance without rerunning tests. After review
   fixes, rerun only directly affected current-ticket selectors. Under `review=off`, do
   not create a reviewer/evaluator.
5. **Deliver once stable.** After focused tests and optional review, regenerate the
   effective-on-merge note and stage its exact path with the ticket's final local commit.
   Push the first stable candidate and open one PR; avoid intermediate CI-triggering
   pushes. A repair after target movement may require another stable-candidate push.
   Allow each mandatory hook, merge command, branch-protection and CI mechanism to run
   once per candidate; do not bypass them or duplicate checks on the same inputs. Run the
   merge-time static gate below before merging. Merge only when it, current-ticket
   evidence and required protections are green.
6. **Finalize, then advance.** Confirm the merge, atomically mark the ticket `done`, clear
   the active ticket and derive the new frontier in external state. Synchronize the base,
   confirm the projected note landed with the PR and select the next ticket. Do not create
   a follow-up note commit.

## 5. Focused development test policy

- Run only explicit test cases/files/selectors that prove the current ticket's acceptance
  criteria or directly changed behavior. Targeted lint, typecheck or build checks may run
  when the ticket or project requires them; they do not broaden test selection.
- Do not manually run repository-wide, package-wide or full regression *test* suites.
  Do not run tests from earlier tickets merely because they ran before; include one only
  when the current ticket changes the behavior it exercises. Section 6 separately
  requires broad static checks on the integrated merge candidate.
- Never manually repeat a passed test on unchanged relevant inputs. After a code change,
  rerun only selectors whose behavior or inputs changed. After a failure, diagnose and
  rerun the failed or invalidated selector, not every current-ticket test.
- If no focused selector exists, add one when the ticket calls for test coverage;
  otherwise use the narrowest supported selector and record the limitation. Never fall
  back to a full suite merely because selection is inconvenient.
- Broader mandatory hooks or CI may run automatically. Let each mechanism run once for a
  stable candidate and reuse its result when it covers the same inputs; a PR-head result
  does not replace a merge-candidate result. If project policy separately requires direct
  invocation of a full regression *test* suite, report the conflict instead of running
  it; a delivery command whose hook launches one is an automatic side effect, not a
  direct invocation.
- Record selector, purpose, outcome, duration, tested head and relevant input fingerprints
  in a compact receipt. Never call a skipped or inherited result freshly passed.

## 6. Merge-time static gate

After the PR opens and before merging into its target (`develop` when that is the target),
the orchestrator fetches the latest target and PR heads and validates their integrated
result. Record both head SHAs and the candidate's tree; green checks on the PR head alone
or on an older target are insufficient.

Run the project's configured repository-wide static checks across all applicable
packages/workspaces, not just those changed by this ticket: package or lockfile integrity
checks, lint, TypeScript typechecks and required build/compile or other static checks. Use
an up-to-date merge-queue/merged-result CI gate when it covers those checks on that exact
candidate. Otherwise prepare the result of the planned merge method in an isolated
worktree/ref and run the missing checks there. Do not reinstall dependencies or force a
clean build unless the checks require it, and do not manually duplicate equivalent
checks already green on the same candidate. This gate does not add a manual full
regression *test* suite.

An integration conflict or failed check blocks the merge. Repair on the ticket branch,
rerun invalidated focused tests, and validate the new integration result. If the target
advanced, also reconcile any changed tracked project note or ticket-relevant instructions
against the new base; never merge a stale projection. If either head changes before merge,
discard stale gate receipts and repeat the full applicable static gate on the new
candidate. Merge only against the validated target/head pair, using a merge queue,
up-to-date protection or an equivalent conditional merge to prevent a race. If none is
available without bypassing protections, stop for user direction. Confirm the landed tree
matches the validated result; if it does not, run the static gate on the landed tree and
do not advance the queue while it is red. Keep gate receipts with the two heads, candidate
tree, check coverage and outcomes in external state.

## 7. Stop

Continue while any eligible ticket remains. Stop when the pool is drained or every
remaining ticket is blocked on a genuine product decision, policy conflict or external
dependency. Difficulty, a test failure or an inferable implementation choice is work to
resolve, not skip. Persist blockers and affected dependents; comment on the tracker only
when authorized. Report the queue result, tracked note path, reusable learnings and any
newer queue state that could not be published because no implementation PR existed.

Keep feature pushes, required protections and the user's scope intact throughout the loop.

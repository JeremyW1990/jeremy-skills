---
name: loop-tickets
description: "Complete a dependency-ordered ticket pool on one branch, with checked commits per ticket and one final PR, resumable state, repository-local project notes, project-authorized focused testing, and optional code review. Use after to-tickets or when asked to work a ticket queue end to end."
---

# Loop Tickets

Work one implementation ticket at a time on one pool branch. The orchestrator owns pool
state, ticket selection, optional review and final PR delivery; each ticket worker owns
implementation and focused feedback. Use `$loop-tickets` in Codex and `/loop-tickets` in
Claude Code. Follow the user's authorization for publishing and merging; no option
permits deployment.

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

Persist the pool's exact sources/versions, target remote/branch, starting commit, branch,
worktree, last checked commit, final PR and integration status. For each ticket, persist
its title, acceptance criteria, blockers, worker, starting commit, resulting commits and
receipts. Ticket statuses are `todo`, `working`, `validating`, `committed`, `blocked` and
`done`: `committed` means acceptance and required commit checks passed on the pool branch;
`done` requires the pool's confirmed merge. Keep final integration status at pool level
so a failed final gate cannot erase already committed ticket progress.

Derive the active ticket, counts and frontier from ticket statuses on every read rather
than trusting cached summaries. Validate dependency closure and cycles once, and
batch-fetch the pool. A checked, committed ancestor on this pool branch satisfies an
ordinary implementation dependency within the pool. An explicit requirement for an
already merged, deployed or externally delivered dependency still requires that actual
state; do not silently weaken it or split the pool to work around it.

Persist only redacted identifiers, compact receipts and evidence pointers. Never record
secrets, credential-bearing URLs, environment values, tokens, customer data or fixture
payloads.

## 2. Start once, or resume the existing branch

For a new pool, fetch the latest `develop` from the repository's GitHub remote once and
create one pool branch/worktree from that exact commit. If the authoritative queue names
another target, use that exact branch instead. Record its identity before starting the
first worker. Reuse this branch and worktree for every ticket; do not pull the target,
create ticket branches or open ticket PRs between tickets.

On resume, locate the saved pool branch/worktree and verify its history against external
state and receipts. Preserve uncommitted work and reconcile interrupted commits or
checks before advancing. Resume on its accumulated code; never reset it to develop or
create a fresh branch just because this is a new session. Refresh the target again only
at final integration, unless the user explicitly changes the workflow.

## 3. Maintain the note and prepare only what the ticket needs

The note lives at `<pool-worktree-root>/.loop-tickets/project-notes.md` and travels with
the pool's implementation commits and final PR. Authoritative queue state remains
outside the repository. Before creating or updating the note, read
[references/project-notes.md](references/project-notes.md). Read the initial target's
note at pool start and the existing pool note on resume.

Each ticket's final local commit includes a cumulative note distinguishing committed
work from merged work. The last ticket instead prepares the pool's final
`effective-on-merge` snapshot. Neither snapshot invents future check results, PR URLs or
merge identities. Reconcile it if the target advances during final integration. Never
create a note-only branch, commit, push, PR, CI run or test; fold note changes into the
implementation or integration commit. A required note failure blocks that commit unless
the user waives it. Lessons do not alter the approved pool or become project policy.

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

## 4. Commit tickets in dependency order

Resume an active ticket first. Otherwise choose an eligible `todo` ticket whose blockers
are satisfied. Follow authoritative priority; use ticket number as the final tie-breaker.

1. **Capture the ticket boundary.** Record the pool branch's current commit as this
   ticket's starting point. Earlier tickets remain committed ancestors on the same branch.
2. **Use one worker per ticket.** In Codex use `spawn_agent` with `fork_turns: "none"`;
   otherwise use the host equivalent. Reuse the same worker for all repairs on that
   ticket. Pass the shared branch and local-commit boundary to the runner; any runner
   defaults for separate branches or PR delivery yield to this pool workflow. Only the
   orchestrator changes queue state or performs final delivery.
3. **Implement with authorized focused tests.** The worker runs only the smallest
   selectors proving the current ticket when project instructions and the current
   conversation authorize tests; otherwise it records them as unrun and inspects the
   diff to prepare a stable candidate. It does not run a full regression suite, a broad
   final gate, a separate review or an intermediate push.
4. **Review if enabled.** Under `review=on`, the orchestrator invokes the configured
   project reviewer or `code-review` once on this ticket's diff from its recorded starting
   commit, including its uncommitted changes. Do not review all preceding pool commits
   again. Supply existing test receipts or unrun status and request standards/spec review
   without test execution. After fixes, rerun only directly affected current-ticket
   selectors when authorized. Under `review=off`, do not create a reviewer/evaluator.
5. **Commit through the repository workflow.** After focused tests or a recorded skip and
   optional review, stage the implementation and required note. Let the repository's
   before-commit hook run on every commit and observe its result as described in section 6.
   Do not bypass it or manually duplicate its checks. A required acceptance criterion
   that cannot be established without an unauthorized test remains a blocker.
6. **Record checked progress, then advance.** Confirm the commit exists and its required
   checks passed for the committed candidate. Atomically record the ticket `committed`,
   its commits and receipts, then clear the active ticket and derive the new frontier.
   Do not mark it merged/done or close its tracker issue yet. Continue on the same branch
   without pulling develop, pushing or opening a PR. Intermediate pushes require an
   explicit user instruction.

If a ticket needs several commits, every commit follows the repository's commit checks;
only sufficient acceptance evidence and the final checked candidate permit advancement.

## 5. Focused development test policy

Project and current-conversation test authorization takes precedence over this skill.
When project policy requires explicit execution authorization, a ticket asking for test
coverage authorizes writing tests, not executing them.

- When tests are authorized, run only explicit cases/files/selectors that prove the
  current ticket's acceptance criteria or directly changed behavior.
- Do not manually run repository-wide, package-wide or full regression *test* suites.
  Do not run tests from earlier tickets merely because they ran before; include one only
  when the current ticket or final integration changes the behavior it exercises.
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

## 6. Observe the two repository gates and deliver the pool

The repository owns check commands, scope, timing, hooks and CI/CD configuration. This
skill does not define or configure them, add a separate static gate or manually repeat
repository-managed checks. It observes two lifecycle results: checks before each commit
and the repository's final gate before merging the pool PR into the target. The latter
may be implemented by a repository merge command or remote workflow; do not assume a
local Git merge hook governs GitHub PR merges.

Record candidate identities, required outcomes and compact evidence pointers in external
state. Accept only results the repository considers valid for the current candidate.
Missing, pending, failed, cancelled or stale results are not success. Absence of remote
checks does not replace required local evidence.

During ticket work, keep the current ticket active while checks are unsatisfied. Send a
failed check back to the same worker for diagnosis and repair, preserving the branch and
work. Let the normal repository workflow check the repaired commit; rerun invalidated
focused tests only when authorized. Do not advance to another ticket. If execution is
externally blocked, persist the blocker and stop until it can resume.

After every ticket is `committed`, enter final integration on the same pool branch:

1. Push the stable pool candidate and open one PR for the entire pool, or reuse its saved
   PR on resume. Establish this PR before invoking a repository merge entrypoint that
   requires its number. Never publish a partial pool unless the user explicitly requests it.
2. Follow the repository's supported merge workflow to fetch the latest target and
   integrate any target movement on the same pool branch. When its merge helper owns
   refresh, integration or pushing the updated candidate, let it perform those steps;
   do not duplicate them manually. If it requires a branch repair first, resolve
   conflicts and reconcile changed instructions and the note, preferably with the relevant
   retained worker. Every new integration or repair commit must pass the repository's
   before-commit checks, then update this same PR through the supported workflow. A clean
   automatic merge alone does not prove that the combined code passed its required checks.
3. Observe the before-merge result produced by that workflow. It must establish that the
   final candidate and current target satisfy required checks and protections. Let the
   repository decide when prior results remain valid; do not repeat static checks merely
   because a PR is merging. If further target movement invalidates the candidate, repeat
   the necessary integration and checked-commit steps on this branch and update this PR
   before retrying. Never bypass freshness protections to force a stale merge.
4. Keep a failed or pending final gate in the pool's integration stage. Preserve the
   branch, PR, committed-ticket records and relevant workers; repair without starting a
   new pool or declaring tickets delivered. If the failure invalidates a ticket's
   acceptance evidence, reopen that ticket for repair within this same pool.
5. Confirm the PR actually merged and any repository-required merge verification
   completed. Only then mark the pool and its tickets `done` in external state and
   confirm the final projected note landed. Close tracker issues only when authorized.
   Do not create a follow-up note commit.

## 7. Stop

Continue until all tickets are checked and committed, then complete final integration;
exhausting the implementation frontier alone does not complete the pool. Unsatisfied
required checks retain the current ticket or final integration stage for repair or
waiting. Stop when delivery completes, execution is externally blocked, or every
remaining ticket is blocked on a genuine product decision, policy conflict or external
dependency. Difficulty, a check failure or an inferable implementation choice is work to
resolve, not skip. Persist blockers and affected dependents; comment on the tracker only
when authorized.

Report committed progress separately from merged delivery, the pool branch/PR, tracked
note path, reusable learnings and any unpublished state. A blocked pool stays on its
existing branch for resume; do not deliver a partial subset without user direction.

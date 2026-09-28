# jeremy-skills

Codex and Claude Code skills, kept in one place so they can be installed on any machine
and used across projects. Use `$skill-name` in Codex and `/skill-name` in Claude Code.

## Skills

### [`loop-tickets`](skills/loop-tickets) — `/loop-tickets`

The durable loop between `/to-tickets` and a ticket runner.

`/to-tickets` breaks a plan into tracer-bullet tickets with their blocking edges, and a
ticket runner such as `/implement` or `/tdd` builds one. Nothing drove the pool.
`loop-tickets` is that: validate the graph, start one branch from the latest target,
implement and commit tickets in dependency order, then open one pool PR and follow the
repository's workflow to integrate the latest target and merge.

It adds only what running *many* tickets needs and running one does not:

- the pool written to disk, so a compaction cannot lose it
- a tracked `.loop-tickets/project-notes.md`, so every project shares its queue lessons
  on GitHub through the pool PR
- dependency-closure validation, so a missing blocker cannot strand the run later
- a fresh context per ticket, so ticket N+1 does not inherit ticket N's
- one worker retained through each ticket's repairs, so an attempt does not pay repeated
  discovery and handoff costs
- ticket-owned persistent test data when needed, without resetting a shared database
- focused per-ticket development tests, without manually repeating earlier-ticket or
  full-regression suites
- conditional UI guidance before implementation, then focused UI/UX review and actual
  Playwright acceptance with inspected screenshots when user-visible behavior changes
- observation of repository-managed commit and final merge checks, keeping failures on
  the current ticket or pool integration stage without defining CI/CD logic or duplicating
  checks
- categorized product and dependency blockers; checked commits can satisfy ordinary
  dependencies within the pool, while explicit merged or deployed prerequisites still
  require those states

The entire pool uses one branch and one final PR. Each ticket leaves checked commits on
that branch; no per-ticket target refresh, push, PR or merge is needed. Resume keeps the
existing branch and accumulated progress. A blocked pool is not published in part unless
the user explicitly requests it.

It defaults to `/implement`, but records and honours an explicitly selected user or
project runner. It therefore still fits the
[mattpocock skills](https://github.com/mattpocock/skills) chain
`to-spec → to-tickets → loop-tickets → implement → code-review` without forcing that chain
on a queue with a different configured workflow.

Loop state lives outside the working repository, under the host's
`loop-tickets/<repo-key>/<pool>/` directory: `$CODEX_HOME` (default `~/.codex`) for
Codex, or `~/.claude` for Claude Code. The compact note instead lives at the tracked
repository path `.loop-tickets/project-notes.md`. Each ticket includes a cumulative
committed-but-unmerged snapshot with its final implementation commit; the last ticket
prepares the pool's effective-on-merge snapshot. The note summarizes progress, the
frontier, validation cost, repeated waste and future ticket-design lessons without
note-only commits or pushes. External state records actual check and delivery results.
Existing matching state is reused on resume.

Code review defaults on. Tickets with user-visible UI or interaction changes also use
`ui-ux-pro-max` before implementation against the approved design, then focused UI/UX
review and Playwright browser acceptance on the affected pages, states and viewports.
Valid implementation evidence can be reused. `review=off` omits only the separate code
review; required UI acceptance still applies under the project's test authorization.

To omit the separate code-review pass:

```text
$loop-tickets review=off
```

During development, the loop runs only the smallest local test selectors related to the
current ticket; it does not manually run full regression suites or tests from earlier
tickets without a reason in the changed behavior. At pool start, it fetches the target
branch (`develop` by default) once. Once all tickets are committed, it opens one pool PR
and follows the repository's merge workflow to refresh and integrate the target on the
same branch. It observes repository-owned checks before
each commit and before the final PR merge. Failed commit checks stay on the current
ticket; failed final checks retain the pool's integration state. Tickets are only marked
done after the pool PR has merged. The loop does not define CI/CD logic, bypass
protections or duplicate checks.

### [`implement`](skills/implement)

The companion runner follows the selected verification/review policy, uses focused
development tests, and leaves PR delivery to the queue. `review=off` skips the separate
review stage without dropping current-ticket acceptance requirements.

### [`tdd`](skills/tdd)

Behavior-focused testing at approved boundaries. Ticket-approved seams need no repeated
confirmation; persistence, authorization and rollback can be explicit database contracts.
Necessary local cleanup does not require a separate Code Review stage.

## Install

Symlink the skills you want into `~/.claude/skills/`, so a `git pull` updates them in
place:

```sh
git clone https://github.com/JeremyW1990/jeremy-skills.git ~/.local/share/jeremy-skills
mkdir -p ~/.claude/skills
ln -s ~/.local/share/jeremy-skills/skills/loop-tickets ~/.claude/skills/loop-tickets
```

For Codex, use `~/.codex/skills/` (or `$CODEX_HOME/skills/`) as the link directory.
The same installation approach applies to the companion `implement` and `tdd` skills.

If a directory of that name already exists, move it aside first — `ln -s` against an
existing directory nests the link inside it rather than replacing it:

```sh
[ -e ~/.claude/skills/loop-tickets ] && mv ~/.claude/skills/loop-tickets ~/.claude/skills/loop-tickets.bak
```

Verify:

```sh
test -L ~/.claude/skills/loop-tickets && readlink ~/.claude/skills/loop-tickets
```

A skill at `~/.claude/skills/<name>/SKILL.md` whose frontmatter `name` matches its
directory becomes `/<name>` in Claude Code. `loop-tickets` allows model invocation so an
explicit request to drain a queue can keep running across tickets; that does not broaden
the user's authorization for pushes, merges, deployments, or other external mutations.

## Conventions

- One directory per skill under `skills/`, matching the frontmatter `name`.
- `SKILL.md` carries the control flow; long reference material goes in `references/`,
  which the skill reads on demand.
- Skills are project-agnostic. Anything project-specific is detected at run time and
  recorded as a binding, never hard-coded.

## License

MIT — see [LICENSE](LICENSE).

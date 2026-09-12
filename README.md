# jeremy-skills

Codex and Claude Code skills, kept in one place so they can be installed on any machine
and used across projects. Use `$skill-name` in Codex and `/skill-name` in Claude Code.

## Skills

### [`loop-tickets`](skills/loop-tickets) — `/loop-tickets`

The durable loop between `/to-tickets` and a ticket runner.

`/to-tickets` breaks a plan into tracer-bullet tickets with their blocking edges, and a
ticket runner such as `/implement` or `/tdd` builds one. Nothing drove the pool.
`loop-tickets` is that: validate the graph, take the next implementation ticket whose
blockers are done, branch, give it fresh test data, run the selected workflow, verify
once, merge, and take the next.

It adds only what running *many* tickets needs and running one does not:

- the pool written to disk, so a compaction cannot lose it
- a tracked `.loop-tickets/project-notes.md`, so every project shares its queue lessons
  on GitHub through the same ticket PRs
- dependency-closure validation, so a missing blocker cannot strand the run later
- a fresh context per ticket, so ticket N+1 does not inherit ticket N's
- one worker retained through each ticket's repairs, so an attempt does not pay repeated
  discovery and handoff costs
- ticket-owned persistent test data when needed, without resetting a shared database
- current-ticket-only local tests, so earlier-ticket and full-regression suites are not
  repeated manually
- categorized blocker handling, so one undecidable ticket does not halt a runnable frontier

Each implementation ticket gets its own branch and PR. Project configuration or
authoritative ticket text may explicitly state that one implementation satisfies multiple
issues; otherwise tickets are never combined for convenience.

It defaults to `/implement`, but records and honours an explicitly selected user or
project runner. It therefore still fits the
[mattpocock skills](https://github.com/mattpocock/skills) chain
`to-spec → to-tickets → loop-tickets → implement → code-review` without forcing that chain
on a queue with a different configured workflow.

Loop state lives outside the working repository, under the host's
`loop-tickets/<repo-key>/<pool>/` directory: `$CODEX_HOME` (default `~/.codex`) for
Codex, or `~/.claude` for Claude Code. The compact note instead lives at the tracked
repository path `.loop-tickets/project-notes.md`. Each ticket updates it once as an
effective-on-merge snapshot and includes it in the same implementation PR before the
first push, avoiding a note-only commit or duplicate CI. It summarizes merged progress,
the projected frontier, validation cost, repeated waste and future ticket-design lessons.
Existing matching state is reused on resume.

Code review defaults on. To omit the separate review pass:

```text
$loop-tickets review=off
```

The loop runs only the smallest local test selectors related to the current ticket. It
does not manually run full regression suites or tests from earlier tickets without a
current-ticket reason. Mandatory hooks, branch protections and CI may run broader checks
once; the loop does not bypass or duplicate them.

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

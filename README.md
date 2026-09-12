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
- a local per-project `PROJECT-NOTES.md`, so later pools can reuse verified fast paths
  and avoid recurring failure patterns without turning observations into project policy
- dependency-closure validation, so a missing blocker cannot strand the run later
- a fresh context per ticket, so ticket N+1 does not inherit ticket N's
- one worker retained through each ticket's repairs, so an attempt does not pay repeated
  discovery and handoff costs
- ticket-owned persistent test data when needed, without resetting a shared database
- explicit validation ownership and fingerprinted receipts, so checks already run by a
  hook or CI are not repeated manually on unchanged inputs
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
Codex, or `~/.claude` for Claude Code. A compact `PROJECT-NOTES.md` beside the pool
directories summarizes the live frontier, verified validation facts, repeated waste and
future ticket-design lessons. It is derived and advisory; current tickets, project
instructions and code remain authoritative. Existing matching state is reused on resume.

For a project that chooses local verification and no separate review pass:

```text
$loop-tickets runner=implement validation=local review=none
```

This retains ticket acceptance tests and required remote checks. Preparation and
dependency setup are reused while their fingerprints remain valid; base synchronization
happens once per ticket boundary; and each required check has one primary evidence owner
across focused tests, hooks, local gates and CI. `noMerge` (legacy alias: `dryRun`)
validates and opens the current PR, then stops without merging.

### [`implement`](skills/implement)

The companion runner follows the selected verification/review policy, uses focused
development tests, and leaves the final gate to the queue when the project owns it.
`review=none` skips the separate review stage without dropping acceptance requirements.

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

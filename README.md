# jeremy-skills

Claude Code skills, kept in one place so they can be installed on any machine and used
across every project.

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
- dependency-closure validation, so a missing blocker cannot strand the run later
- a fresh context per ticket, so ticket N+1 does not inherit ticket N's
- new test data per ticket, on a database that is never reset between them
- final-gate coverage inspection, so checks already run by a hook or CI are not repeated
  manually on unchanged inputs
- a skip-and-count rule, so one undecidable ticket does not halt the queue

Each implementation ticket gets its own branch and PR. Project configuration or
authoritative ticket text may explicitly state that one implementation satisfies multiple
issues; otherwise tickets are never combined for convenience.

It defaults to `/implement`, but records and honours an explicitly selected user or
project runner. It therefore still fits the
[mattpocock skills](https://github.com/mattpocock/skills) chain
`to-spec → to-tickets → loop-tickets → implement → code-review` without forcing that chain
on a queue with a different configured workflow.

Loop state lives in `~/.claude/loop-tickets/<repo>/`, never in the repository being worked
on, so the working tree stays clean.

## Install

Symlink the skills you want into `~/.claude/skills/`, so a `git pull` updates them in
place:

```sh
git clone https://github.com/JeremyW1990/jeremy-skills.git ~/.local/share/jeremy-skills
mkdir -p ~/.claude/skills
ln -s ~/.local/share/jeremy-skills/skills/loop-tickets ~/.claude/skills/loop-tickets
```

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

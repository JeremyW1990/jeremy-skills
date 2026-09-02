# jeremy-skills

Claude Code skills, kept in one place so they can be installed on any machine and used
across every project.

## Skills

### [`loop-tickets`](skills/loop-tickets) — `/loop-tickets`

The loop between `/to-tickets` and `/implement`.

`/to-tickets` breaks a plan into tracer-bullet tickets with their blocking edges, and
`/implement` builds one. Nothing drove the pool. `loop-tickets` is that: take the next
ticket whose blockers are done, branch, give it fresh test data, run `/implement`, merge,
take the next.

It adds only what running *many* tickets needs and running one does not:

- the pool written to disk, so a compaction cannot lose it
- a fresh context per ticket, so ticket N+1 does not inherit ticket N's
- new test data per ticket, on a database that is never reset between them
- a skip-and-count rule, so one undecidable ticket does not halt the queue

**Requires** `/implement`, which calls `/tdd` and `/code-review` itself — the
[mattpocock skills](https://github.com/mattpocock/skills) chain it slots into:
`to-spec → to-tickets → loop-tickets → implement → code-review`. A project with its own
`/implement` gets that one instead, which is the intent.

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
directory becomes `/<name>` in Claude Code. `loop-tickets` sets
`disable-model-invocation: true`, so it runs only when you type it — it opens and merges
pull requests, which is not something an agent should start on its own initiative.

## Conventions

- One directory per skill under `skills/`, matching the frontmatter `name`.
- `SKILL.md` carries the control flow; long reference material goes in `references/`,
  which the skill reads on demand.
- Skills are project-agnostic. Anything project-specific is detected at run time and
  recorded as a binding, never hard-coded.

## License

MIT — see [LICENSE](LICENSE).

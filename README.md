# jeremy-skills

Claude Code skills, kept in one place so they can be installed on any machine and used
across every project.

## Skills

### [`loop-tickets`](skills/loop-tickets) — `/loop-tickets`

Works a pool of tickets to completion, one at a time.

It is the downstream half of [`to-tickets`](https://aihero.dev/skills-to-tickets):
`to-tickets` breaks a plan into tracer-bullet tickets with their blocking edges, and
`loop-tickets` then drives them — one ticket, one branch, one PR, one merge, each in a
fresh context, each with its own newly created test data.

Three things distinguish it from a plain "do these in order" loop:

- **The pool is the conversation, not a tracker query.** A tracker search returns whatever
  else is open — bug reports, debt notes, parent specs — and cannot tell a ticket you just
  drafted from an issue that has been sitting there for months. The tickets are captured to
  disk before the first one starts, so a context compaction cannot lose them.
- **It composes rather than reimplements.** The loop hands each ticket to `/implement`,
  which calls `/tdd` and `/code-review` itself. This skill owns only what `/implement` has
  no place for — the things that go wrong *between* tickets rather than inside one.
- **The database is never reset between tickets.** Each ticket adds its own namespaced
  fixtures and reads the accumulated corpus freely; nothing a ticket did not create is
  mutated or deleted. An empty database measures nothing, and a bulk prefix sweep is a
  database reset in disguise. See `references/test-data.md`.

All loop state lives in `~/.claude/loop-tickets/<slug>/`, never in the repository being
worked on, so the working tree stays clean.

**Requires** `/implement` (and the `/tdd` and `/code-review` it calls) — the
[mattpocock skills](https://github.com/mattpocock/skills) chain it slots into:
`to-spec → to-tickets → loop-tickets → implement → code-review`. A project with its own
`/implement` gets that one instead, which is the intent.

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

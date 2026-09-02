# Per-ticket test data, on a database that is never reset

## The rule

> **Each ticket creates its own namespaced test data, and reads the accumulated corpus
> freely. Nothing a ticket did not create may be mutated or deleted.**

That is the whole protocol. It is add-only: ticket N+1 does not inherit ticket N's
fixtures as *its* fixtures, and it does not delete them either.

**Why the corpus accumulates.** An empty database measures nothing. The defects worth
catching are the ones only a real population reveals — a column whose every stored array
happens to hold exactly one element, which licenses a false invariant nobody would think
to question; a field every live row turns out to carry, which stops a redesign from
deleting the only place a value lives; a declared numeric precision that would silently
round every stored amount. None of those are visible on a clean database, because a
clean database has no population to be wrong about.

**Why the fixtures are fresh.** A ticket that asserts against another ticket's rows is
measuring an artifact it does not control, which moves under it when that ticket is
revised. Fresh, namespaced rows make the assertion stable and make the ticket's own
footprint recoverable later.

**The two are not in tension** — they are different verbs. A ticket **writes** new,
tagged data. A ticket **reads** everything. Conflating the two is what makes the rule
look contradictory.

**Reset only** when the ticket touches a migration, a policy/RLS file, or the schema
definition. Then build a **uniquely-named scratch database**, run migrations from zero,
reapply policies, and drop it precisely afterwards. Use the project's own reset command,
never one synthesized from framework defaults — a project wrapper usually exists precisely
because the framework default is broken for that project. A common shape: the default
reset recreates only the primary schema, so a project whose migrations create their own
schemas dies partway through the replay with a misleading error.

**From-zero construction still has to happen somewhere.** It is the only proof that the
migration chain applies in order on a clean database. Run it as its own periodic gate,
not as a per-ticket step. Skipping it entirely means finding out at deploy time.

---

## Namespace

Mint one per ticket, before creating anything:

```sh
ns="<ticket-id-lower>-$(date +%s)-$(openssl rand -hex 4)"
```

Stamp it on **every identity-bearing value**: tenant/org names, every email address,
trace and correlation ids, session and run ids, generated filenames, temp directories,
queue names, and any record created in an external service.

**Two marker levels, because they answer different questions.** A reserved domain on
every fixture email (`@example.com` or equivalent) answers *"is this residue?"*. The
`<ticket>-<nonce>` answers *"is this MINE?"*. Keep both.

**The nonce is load-bearing for concurrency, not just for re-runs.** Sibling agent lanes
routinely share one local database. A sweep keyed on a bare `<ticket>-*` glob would
delete another lane's in-flight fixtures for the same ticket. **Sweep keys come from the
manifest, never from a prefix glob.**

**Seed through the product's canonical writers**, never by direct column insert. A
hand-inserted projection has no audit row and no derived records, so it is evidence of
nothing. Re-running a seeder should ADD a fresh cohort rather than edit an existing one.

---

## Manifest

Nothing like this exists in most projects; it is the loop's own artifact. Write it **at
creation time**, on disk, outside the repository, so it survives compaction and a
container swap. Leaked fixtures accumulate across separate sessions precisely because
nothing recorded what each run created.

`~/.claude/loop-tickets/<slug>/fixtures/<ticket-id>.json`:

```json
{
  "ticket": "07",
  "namespace": "t07-1756800000-3f9c1a2b",
  "reservedDomain": "example.com",
  "startedAt": "2026-09-02T01:40:00Z",
  "seedCommand": "<the exact command run, or null>",
  "createdIn": ["organizations", "clients", "documents"],
  "artifacts": ["/tmp/…"],
  "durable": false,
  "sweep": { "command": null, "safety": "by-care", "verified": null }
}
```

Every field has a reader — a field nothing reads is decoration:

| field | who reads it |
|---|---|
| `ticket`, `startedAt` | the loop, to age and attribute a cohort |
| `namespace` | the class-C sweep key — the only sanctioned one |
| `reservedDomain` | the coarse "is this residue?" screen |
| `seedCommand` | re-running the cohort, and the PR body |
| `createdIn` | tells the post-condition query which tables to re-check |
| `artifacts` | the no-database mode's sweep list |
| `durable` | the sweep's refusal test |
| `sweep.command`, `sweep.safety` | chooses the class-A/class-C treatment |
| `sweep.verified` | the loop's completion criterion for the sweep |

- **`durable: true`** marks data seeded to satisfy a non-vacuity gate. That data *is*
  corpus, and the sweep refuses to touch it. This single field is what would have
  prevented a bulk sweep from deleting the evidence other suites depend on.
- **`sweep.verified`** is set only after an independent post-condition re-query returns
  zero — never by trusting a teardown hook.

---

## Detecting how this project makes test data

Run the ladder; stop at the first that answers, but check all four for shape.

```sh
# 1 — a declared config wins, if the project ships one
#     (no standard key exists; look for a testData block in whatever config the repo has)
ls .claude/*.json 2>/dev/null | while read f; do jq -r '.testData // empty' "$f" 2>/dev/null; done

# 2 — seed / reset / fixture scripts
python3 - <<'PY'
import json, glob
for p in glob.glob('**/package.json', recursive=True):
    if 'node_modules' in p or '.claude/worktrees' in p: continue
    for k, v in (json.load(open(p)).get('scripts') or {}).items():
        if 'seed' in k or 'reset' in k or 'fixture' in k: print(p, k, v)
PY

# 3 — fixture / factory surfaces
find . -type d \( -name fixtures -o -name factories -o -name __fixtures__ \) \
  -not -path '*/node_modules/*' -not -path '*/.claude/worktrees/*'

# 4 — is there a database at all?  (find, never ** — see below)
find . -maxdepth 4 \( -name schema.prisma -o -name alembic.ini -o -name 'migrations' \
  -o -name 'schema.rb' -o -name '*.sqlite*' \) \
  -not -path '*/node_modules/*' -not -path '*/.claude/worktrees/*' 2>/dev/null
grep -rlE 'testcontainers' --include='*.ts' --include='*.py' . 2>/dev/null | head -1
```

**Never write `**` in a command an agent may run.** macOS ships bash 3.2, where
`globstar` does not exist and cannot be enabled, so `**` silently collapses to `*` and a
two-level-deep schema is missed entirely. The agent then reads "no database", drops the
seed and sweep steps, and the whole protocol disables itself without saying so. Use
`find`. Exclude worktree copies from every scan, or it reports phantom duplicates.

**A negative result here disables the protocol, so it must be a positive finding of
absence.** Record the exact command and its empty output in the manifest before
concluding a project has no database.

**Classify a candidate command by READING it, never by running it — not even with
`--help`.** A shell script with no argument parser ignores the flag and executes anyway.
A reset wrapper that drops schemas and then runs a framework reset will treat `--help` as
a no-op and proceed, so "confirming" the command *is* running it, and the accumulated
corpus is gone. Read the source instead (`sed -n '1,60p' <script>`) and classify it as
read-only, additive, or destructive. A command whose source contains `DROP`, `TRUNCATE`,
`migrate reset`, `--force-reset` or `rm -rf` is recorded in `bindings.resetDb` and is
never executed outside the scratch-database branch of this document.

Cross-ecosystem signals (`prisma/seed.ts`, `conftest.py`, `spec/factories/`) are
heuristics, not observations — projects routinely place their seed somewhere else.

**Every seed or sweep carries a hard environment guard, not a comment.** Check that the
connection string resolves to a loopback host and the expected database name, and fail
closed. A prohibition that lives only in prose is a comment. If the target cannot be
proven local or scratch, refuse to run.

**Prefer one committed seed script per ticket that needs data**, named for the ticket.
That makes the data reproducible for the next person and keeps the namespace in source.

---

## Sweeping — by class, never by calendar

| Class | What it is | Rule |
|---|---|---|
| **A · orphan-keyed** | Rows whose parent no longer exists | **Sweep freely**, at every ticket boundary. Safe by definition rather than by care, and idempotent. Delete children-first over a declared ordered list, never a guessed order. |
| **B · schema & session residue** | Scratch tables and their grants, disabled triggers, leaked queue rows, global-accumulator poisons, stray worker processes | **Always sweep at ticket end.** No evidentiary value, and disabled triggers silently remove a protection so later tickets get false greens. |
| **C · this ticket's own live rows** | Matching this ticket's exact `<ticket>-<nonce>` **from the manifest** | Sweep **only** if `durable:false`. Never by bare prefix; never by reserved-domain match across the database. |
| **D · everything else** | | **Leave it.** If a corpus-wide census gate is red, print the offending rows first, join the tenant table for contact address and creation time, and ask whether that residue is the **subject** of another gate before deleting anything. |

**A bulk prefix sweep is a database reset in disguise.** Sweeping leaked fixture tenants
by prefix can remove the very rows a whole class of suites asserts the existence of —
non-vacuity guards, which fail when the corpus is empty. Left-behind fixtures often ARE
the corpus those gates measure.

**Derive the delete order from the live catalog, never from a remembered list.** A
recorded sweep recipe in one project is already stale and would fail: the column it keys
on was dropped and the foreign key it blames no longer exists. Query the constraint
catalog at run time, or call the project's own declared teardown.

**Pair every clear with a detect query, and run the detect AFTER the clear.** A clear
that silently matches nothing is worse than none — one recorded recipe compared a date
column to text and cleared nothing while appearing to work.

**Verify by re-querying, never by trusting a teardown hook.** Most `afterAll` hooks wrap
the delete in an empty catch, so a foreign-key violation is swallowed and the fixture
survives. And a cleanup helper has already *caused* the leak it existed to prevent: it
disabled all triggers to delete a parent, and because a cascade **is** a trigger, it
removed the parent and orphaned 43 children. Never blanket-disable triggers as a cleanup
shortcut.

**Never probe with a bare DELETE.** To show a gate is non-vacuous, run the mutation
inside a transaction that throws at the end, or mutate the source and republish.

---

## When there is no database

Many projects have none, and the protocol degrades cleanly:

- **Drop** the seed step, the sweep step, and every database check.
- **Keep** the namespace — it still applies to temp directories, generated files, queue
  names, recorded cassettes and any record created in an external service.
- **Keep** the manifest — it becomes the record of files, directories and external
  records the ticket created.

**Do not trust a claim that a suite is database-free.** Verify empirically before relying
on it: a gate described as "static" commonly contains a minority of files that open a
real connection, and the whole gate then fails on a checkout with no connection string
configured. Run it with the database down and see.

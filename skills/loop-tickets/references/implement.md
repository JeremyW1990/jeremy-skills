# Implement one ticket, end to end

This is the authoritative process for a single ticket. It ships **inside**
`/loop-tickets` rather than pointing at any project's own `implement` skill, so the
same bar applies in every repository.

It exists because process knowledge has to outlive two things that reliably happen
during a long ticket queue: **context compaction** and **container replacement**. If
either has just happened to you, read this file before doing anything else.

**One ticket, one branch, one PR.** The only sanctioned exception is splitting *one
ticket* across two PRs when a self-contained prerequisite is ready before the rest.
Never the reverse — never two tickets in one PR. The reason is attribution, not
policy: interleaved failure classes cost more to untangle than the batching saves.

Where this file says `{{binding}}`, use the value the loop recorded in
`state.json.bindings`. If a binding is absent, that step does not apply.

---

## 0. Before anything: is the environment alive?

Run `{{bootstrap}}` if the project declares one. It is idempotent; re-run it freely.

Confirm the database connection is the **local/scratch** one and that the application
role is *not* a superuser and does not bypass row-level security, if the project has
such a role. Security policies only actually execute on a non-privileged role — two P0
defects in already-merged work were found solely because that role was real, and both
were invisible to every test that ran on the privileged connection.

**Confirm you can reach the issue tracker before you start, and use whichever transport
the loop told you is available.** Both `mcp__github__*` tools and the `gh` CLI can
deliver a PR; a subagent commonly has `gh` and not the MCP tools. Take the transport the
loop named. Only if *neither* is reachable should you stop — without one, finished work
cannot be delivered, so say so loudly rather than continuing to implement. **A transient
"MCP server disconnected" notice is not the same thing**: re-check once (servers usually
reconnect within seconds) before concluding a transport is absent.

Never connect to shared, staging or production systems, not even for a read-only query.
Never add an API key for a live model. Never run suites that call a real provider.

**An env file is not shell-source-safe.** Values routinely contain characters a shell
re-interprets, so `source .env` can mangle a connection string or execute a substitution.
Pass it to the tool that understands it (`--env-file`, `dotenv`, the test runner's own
loader) instead of sourcing it.

**Completion criterion:** the environment check passed and the tracker transport is
named.

---

## 1. Read the authorities, in this order

1. The project's own agent guide — `CLAUDE.md`, `AGENTS.md`, or whatever
   `{{readFirst}}` names. Long, and worth it.
2. The architecture and data-model documents it points at.
3. The program or sprint contract, and the ticket map, if the project has them.
4. The ticket itself: body, checkboxes, comments, blocking edges.
5. Whatever that ticket references — capability registries, decision records,
   subsystem maps.

Where a project marks which parts of a design document are **shipped** versus
**target**, only the shipped part is real. Everything else must not be opened early.

When code, the ticket, the map and the architecture disagree, **verify against the code
and the database**. Do not invent a compatibility layer or a dual-read path to make them
agree.

The strongest single habit: **read the most recent one or two implemented siblings
before designing anything.** They are the shape the reviewer expects, and they encode
decisions no document states.

**Completion criterion:** you can state, in one sentence each, what the ticket changes
and which existing module is its nearest sibling.

---

## 2. Confirm the spec, and that the ticket is still real

- Every direct blocker is closed.
- The approved surface, role, authority and batch cardinality are explicit.
- No unratified product or architecture choice is hiding in the ticket.

**Re-derive the ticket's own measurement before starting.** A ticket body is a snapshot
of the plan on the day it was filed, and the plan moves underneath it. A ticket that
enumerates defects is often part-fixed by the time it is picked up; a ticket that quotes
a measurement often re-measures somewhere else entirely. Either can sit queued as blocked
long after the work stopped being needed.

If the ticket's premise no longer reproduces, the ticket is **closed or rewritten, not
implemented**. Say which half was already done and which half remains.

**Absence of a marker is evidence of nothing.** Read code and tests, never a design doc's
silence: design documents lag the code, so a marker is positive evidence that something
shipped while its absence proves nothing at all. The cheapest probe is to run the ticket's
own acceptance assertion and see whether it already passes.

Do not derive permissions from an adjacent ticket. Do not read an existing page
capability as a chat capability. Do not extend one surface to another because "the
backend service already exists".

If an unresolved choice would **materially change product behaviour**, ask. Otherwise
decide, state the assumption in the PR, and keep building.

**Completion criterion:** the ticket's claim reproduces against current code, or the
ticket has been re-scoped in writing.

---

## 3. Define the public test seam before touching production code

Write down, explicitly:

- the canonical domain command
- the safe input/output DTO
- what stays server-private (ids, revisions, digests, generations, HMACs)
- the post-lock authority recheck
- the exact resulting data
- rollback / no-op / idempotency behaviour
- the UI parity route
- the real-permissions integration seam
- the browser path, if the ticket has a browser-visible surface

**Completion criterion:** each of the above is written down before the first production
edit.

---

## 4. Strict TDD — get a real RED first

Reproduce RED through a **public or production seam**, never a private helper. Then
write the smallest GREEN.

The best RED is a probe of the shipped path. Driving the real route composition — rather
than the handler in isolation — has a habit of proving the shipped path already throws on
a case it claims to support, which no amount of reading surfaces.

Never:

- implement first and backfill tests
- mock away the database, the permission layer, or the worker behaviour the ticket is
  about
- raise a timeout to make a failure disappear
- convert an unknown error into a generic success
- weaken a security boundary to get to green

### When a test fails, decide whether it is yours — the attribution ladder

1. **Compare against the baseline at the same scope.** The loop recorded the base
   branch's failing test *names*. A count is not a comparison: a count can match while
   one failure was fixed and another introduced. Diff the **names**.
2. **Read the failure message class before classifying.** A load flake fails on
   timeouts, deadlocks or foreign rows. A **stale assertion** fails on a specific
   expected value — and that is often a test this very ticket is supposed to update.
3. **Re-run the single file in isolation.** Passing alone and failing under load is a
   flake. The reverse also exists: a file that passes in a full run and fails alone.
4. **If (1) and (3) disagree, run the same selection on both sides.** Scope mismatch
   invalidates attribution.

**Use the baseline the loop already captured.** It was measured on the base branch before
your branch existed, at a named scope, and it records failing test *names*. For anything
inside that scope, no re-measurement is needed.

**To measure a NEW scope, commit and switch branches — never stash, and never copy files
out.** Both of those hand-rolled recipes are broken in ways that silently invert the
answer:

- A `stash push` that matches nothing is a no-op, and the paired `pop` then restores an
  *unrelated* stash into your tree — measured twice, once leaving four conflicted files
  from a months-old stash.
- `git checkout HEAD -- <list>` **aborts the entire operation** when one pathspec is
  unknown to HEAD — and strict TDD's defining first act is adding a new test file, which
  is exactly such a path. It reverts nothing, so the "baseline" run is a run of your own
  tree, every regression you introduced appears on both sides, and the comparison
  concludes that nothing is yours. That is worse than no baseline: it manufactures
  confidence. Copying files to a flat directory fails the same way — basenames collide
  (names like `domain.ts` or `index.ts` recur throughout a large tree) and a deleted file is
  silently resurrected.

The safe form moves the whole tree at once, so new files, deletions and duplicate
basenames are all handled by git:

```sh
git add -A && git commit -m "wip: baseline probe"   # reversible; amend or reset after
git switch <base-branch>
<run the same selection, at the same scope>
git switch -                                        # back to the ticket branch
```

Rebuild any compiled artifact the selection reads before running on either side —
a posture check that reads a compiled constant will otherwise test new artifacts against
old source. Confirm `git status --porcelain` is empty before and after.

**Before attributing a failure to code, check the schema object it relies on exists.**
A local database drifts from its committed migrations; a hand-made index whose name
appears nowhere in the repository once made a correct assertion fail. Query the catalog
for the index or constraint, then grep the repository for what you find. A name present
in the database and absent from the repository is drift — repair it locally and never
open a code ticket for it.

**A failure that reproduces on the baseline is recorded, not fixed here.** Note it in the
PR body with its evidence and move on — do not repair it inside this ticket, and do not
let it be attributed to you.

**Verify the instrument before the product.** This class has produced at least nine
recorded wrong conclusions; in one session five harness defects each manufactured a
plausible product story. Before recording any finding about the product, confirm the
harness passes at all, and name which guard actually fired.

**Completion criterion:** a RED that runs through a public seam, and every other red in
the run classified as yours or the baseline's by name.

---

## 5. Clean cutover

Delete: stale code, obsolete wrappers, low-level public writers that bypass the
canonical service, compatibility branches, dual reads, retired enum values, assertions
and prompt pins.

No unapproved backfill. No "two paths for now". When the ticket merges there is exactly
one canonical production path.

**Reverse stale expectations rather than leaving them standing.** If a test pinned
behaviour this ticket deliberately changes, rewrite the test to pin the new behaviour
and say in its comment why the old claim retired. A test asserting a refusal that was
itself the bug is worse than no test.

**Completion criterion:** grep proves the old path has no remaining public caller.

---

## 6. UI parity

A new capability must not be reachable only through the machine-facing surface.

- The human-facing application needs an approved flow with the same semantics.
- Page and programmatic entry share the **same canonical service**.
- Audit provenance differs per spec — record which entry point wrote the change.
- The UI shows the complete safe preview.
- The UI never renders a database id, opaque reference, HMAC, generation or other
  private coordinate. Round-tripping an opaque digest is fine; displaying it is not.
- Tests wait for hydration — a negative assertion against a blank page is a false pass.

**Check the copy against the behaviour.** Two shipped delete prompts asserted the exact
opposite of what the system did, and no test caught it, because no test read the words.

**Completion criterion:** the same act is reachable from the human surface and both
routes enter one service.

---

## 7. Assert resulting data, never "the function was called"

Whichever apply: stored documents, audit-log rows with actor / source / trace / exact
period, tombstones, selection state, immutable run records and their manifests,
invalidated outputs, outbox rows, queue payload and generation, receipts, retained
evidence, cross-tenant invisibility, and **zero external sink activity**.

Read from **final stored rows**.

**A data-shaped refusal is not the product's answer.** If a case cannot be exercised
because the corpus lacks the facts, seed the data first, re-run, and say plainly which
half you fixed. A fixture that cannot exercise the capability only proves "the product
could not answer", which is indistinguishable from a genuine defect.

**Probe destructive hypotheses inside a rolled-back transaction.** Never use a bare
`DELETE` against shared corpus rows to prove a gate is non-vacuous — one such probe
cascaded to Person-scoped families that no republication restores.

**Completion criterion:** every acceptance criterion is asserted against a stored row.

---

## 8. Verify

### Do the predictable guards FIRST, before running anything

A full verification chain typically gets re-run several times per ticket — not because the
work was wrong, but because **the guards fire one at a time**. Lint passes, typecheck
fails; fix, and a census fails; fix, and an immutability check fails. Each discovery
costs a full chain.

Nearly all of it is predictable from what you touched. Build the diff-shape lookup for
the project once and consult it before the first run. Recurring shapes:

| if the change… | then, before verifying |
|---|---|
| adds a database function or object | register it in whatever allowlist the runtime consults, **and rebuild the package that compiles that constant** — posture tests read the compiled value |
| inserts into a policy file indexed by line | re-key the census that indexes by `file:line`; every later site shifts. Check the flagged line holds the same expression at a new offset before treating it as new |
| adds a migration | run the project's artifact-rebind step, and confirm the new migration **sorts last** — one that sorts into the middle of the applied sequence fails an ordered-prefix check at deploy |
| puts a grant/revoke in a migration | move it to the policy file, if policies deploy after migrations and the runtime role does not exist yet |
| edits an already-applied but **unmerged** migration | clear its recorded checksum and re-run the rebind. Immutability protects **merged** migrations; correcting an unmerged one is legitimate |
| edits an already-**merged** migration | do not. A merged migration is immutable, comments included: deploy tooling checksums the file, so a comment-only edit can refuse the whole migration step |
| drops a database function | remove its grant/revoke from the policy file too. A stale line aborts the whole apply, and every policy after it never installs |

### Then run the independent checks CONCURRENTLY

`{{lint}}`, `{{build}}`, `{{typecheck}}` and the static gate do not depend on each
other. Running them together halves the wall time and, more importantly, surfaces
**every** failure in one pass instead of one per cycle.

### A green pre-push gate is not a green suite

Where a gate selects tests by filename, its membership is an accident of naming — in one
measured repository 108 of 840 test files were gated and 732 were not, and real guards
sat red for weeks because of what they were called. Report a ticket verified on the
gate **plus** the ticket's own suites at module scope, never the gate alone.

### Batch by CLASS, not by call site

A batch of similar conversions in one pull request costs roughly what one does, and the
adjudication is *better* — the shared classes get reasoned through once instead of once
per site. Splitting a mechanical substitution across many pull requests buys nothing and
loses the shared reasoning.

**But scan for the THING ITSELF, not for the class's usual shape.** Scanning for one
comparison form reported a class clean while four sites of the same kind sat in other
shapes — a differently-aliased variant, two local-variable comparisons, and a paired
lineage case. Widen the search to the identifier before believing a class is empty.

### What must not be traded away for speed

- **Mutation probes.** A test nobody has watched fail is decoration. Break the thing the
  test guards and confirm the suite turns red.
- **Reading the code before changing it.** Most real findings come from reading one
  function, not from running one more suite.
- **Attribution.** Never attribute a red to your branch without measuring the baseline at
  the same scope.

### Run at least

- `{{lint}}` and `{{build}}`
- `{{typecheck}}` for every touched package
- focused unit tests
- the project's static security/backstop suite
- real integration against a non-privileged database role
- a from-zero migration + policy apply — **only when the ticket touches a migration, a
  policy file, or the schema definition** (see `test-data.md`)
- resulting-data assertions
- the relevant human-surface tests
- the relevant browser tests — **only when the ticket has a browser-visible surface**.
  Most tickets that add a capability do, because UI parity adds a page flow; this
  exemption is not a licence to skip them
- `git diff --check`
- an independent standards review and an independent spec review

Let the pre-push hook run. Do not bypass it unless you have proven the failure is an
unrelated baseline issue reproducible on the base branch, and recorded the evidence in
the PR.

**Completion criterion:** every check above either passed or is named in the PR body as
a baseline failure with its evidence.

---

## 9. Review — P0 / P1 / P2

Report **specific items** at each level. "P0 0 / P1 0 / P2 0" is not a review.

**P0 — correctness or security. Blocks merge.**
Cross-tenant visibility or a permission bypass; corrupting canonical stored or audit
data; destroying immutable evidence (bytes, lineage, historical records); a real
external side effect escaping (live email, live model call, real queue write); reading
mutable state that decides a write **outside** the serialization lock; hard-deleting
something that must be soft-deleted.

**P1 — violates this ticket's stated acceptance criteria, or the cutover is incomplete.
Must be fixed before merge.**
A surviving dual-read path, compatibility branch, canonical-service bypass or undeleted
stale code; a test that asserts a call instead of resulting data; missing UI parity;
missing no-op / retry / idempotency coverage; a schema change shipped without its
constraint, trigger, policy or grant.

**P2 — does not affect correctness. May merge, but list every item individually in the
PR body.** Naming, comments, test strengthening, readability, accepted residuals.

**Completion criterion:** each level lists items or is explicitly stated empty with the
reasoning.

---

## 10. Ship

**Commit title:** `<ticket-id>: <one sentence, verb first>`. Body explains why, not what.
End with whatever co-author trailer the project uses.

**PR:** open it **ready**, not draft. The body states what changed, how it was verified,
and the itemised P0/P1/P2 review.

**Never put an issue-closing keyword next to an issue reference — not even when
describing one.** The linker matches `close`, `fixes`, `resolves` and their variants
followed by an issue reference anywhere in a PR body, and it ignores negation and
quotation alike.

This has fired twice on the same issue, for two different reasons: a body stating that
the PR did *not* resolve an issue linked it anyway and merging shut the issue; then the
two follow-up PRs that **documented that incident** reproduced the phrase while
explaining it, and shut the same issue again. So "phrase it without the keyword" is not
a sufficient rule — prose *about* the keyword still contains the keyword.

To say a PR leaves an issue open, write "this pull request leaves issue #NNN open". To
describe the trap, refer to the issue number alone and describe the linker's behaviour
without reproducing the pattern. The linker fires on merge only, so an already-merged
body cannot re-trigger; the repair is to reopen the issue and correct anything not yet
merged.

**Merge your own PR** once it is green. Where the loop supplied an `issueRef`, then
confirm that issue reads `CLOSED`; a ticket carrying no `issueRef` is complete at merge.

**"CI green" alone is not sufficient for an unattended merge** — a green PR can be green
because a test was weakened, or can semantically break a consumer no test covers. Before
merging, confirm no test was skipped, deleted or loosened to reach green.

**You cannot ask the human — you have no AskUserQuestion tool.** Where this process would
otherwise pause, hand back instead. If the diff touches schema, migrations, a
policy/RLS file, authentication, money, infrastructure, a CI workflow file, or **deletes a
file**, open the PR ready, do **not** merge, and return
`{"status":"needs-human-merge","pr":<n>,"notes":"<which high-risk path and why>"}`. The
loop holds the decision for the human. The same applies in §2: an unratified product
choice returns `{"status":"blocked","notes":"<the decision, and what answer unblocks it>"}`
rather than being guessed at.

**Keep pushes on feature branches** — integration happens through the PR. Never
force-push, never bypass commit hooks.

### Splitting a ticket

If a self-contained, independently valuable prerequisite is finished long before the
rest, it may ship as its own PR while the ticket stays open — for example a correctness
fix to already-merged work. State plainly in the body that the ticket is not resolved,
and enumerate what remains. Never merge unfinished work under a description implying the
ticket is done.

**Completion criterion:** the PR is merged, the base branch is pulled, and — where an
`issueRef` was supplied — that issue reads `CLOSED`.

---

## 11. Done, and stopping

A ticket is done when the §8 checklist passes. Do not invent extra verification rounds;
do not withhold a finished ticket because it could be better.

**Stop and ask only when:** an unratified choice would materially change product
behaviour · new credentials, permissions or external coordination are required · the
action would clearly widen scope · the same external blocker has recurred with no safe
alternative · the tracker, the database or another required system is persistently
unavailable.

A failing test, a hard implementation, needing more investigation, or a real P0/P1 in
the current ticket are **not** reasons to stop.

### Skipping instead of stopping

A ticket that meets the pause bar may be **skipped** rather than halting the queue:
comment on it, leave it open, and hand back to the loop.

1. **The skip bar is exactly the pause bar** — a product decision the approved material
   genuinely never ruled on. A hard implementation, a failing test, needing more
   investigation, or an ambiguity that can be **inferred from existing precedent** are
   none of them. Prefer inferring and recording the inference.
2. **Every skip comment states three things**: which decision was never ruled on, what
   answer unblocks it, and which downstream tickets it blocks.
3. Do not close the ticket and do not relabel it.

**A worked example of inferring rather than skipping.** A ticket's criterion named two
categories of a thing plus an "ordinary" third that did not exist anywhere in the code.
Rather than skip, the mapping was inferred from precedent: the category constant held
exactly two values, one upload path accepted an enum over both, the other a literal of
one. So the narrower role reaches one category and the wider role reaches both. The
inference was recorded, together with the note that the closure is an application-layer
constant rather than a database enum — a known site to revisit if a third category ever
appears. That is the shape: name the precedent, state the inference, flag where it would
break.

---

## Appendix — fourteen recurring failure modes

Check every one before opening the PR.

1. **Over-coarse revisions.** A global or whole-book revision is not a substitute for a
   per-resource, per-timeline, per-period or per-exact-resource revision. Test
   `add→remove`, `set→clear`, `replace→restore`, `empty→nonempty→empty`. Equal final
   values do not mean unchanged state.
2. **Opaque reference leakage or reuse.** Server-minted, random, attempt-scoped,
   **tenant-bound, subject-bound and resource-bound** — all three axes, not one. Never
   derivable from a database id, never reused across attempts or subjects, never present
   in final prose, logs, errors, receipts or UI.
3. **Subject ambiguity.** Unicode NFKC, whitespace folding, case folding,
   human-distinguishable label checks. If two objects are not safely distinguishable to
   a human, fail the whole batch closed. Never select by array position, ordinal, a
   synthesised name, a caller-supplied raw reference, or a `[0]` default.
4. **Wrong surface.** Separate audiences have separate routers and separate auth
   surfaces. Never share a procedure across them, and never conclude a capability is
   available everywhere because a backend service exists.
5. **Missing first-data initialization.** Test a brand-new subject, an empty container,
   no history, the first exact-period write, the first resource write, and the first
   proposal / review graph / inbox / outbox row. Never assume a container already exists.
6. **Testing the code path instead of the data.** See §7.
7. **Batch atomicity errors.** No partial effect inside one target transaction. Each
   target atomic and independent; one target's conflict does not roll back another's
   success; one subject never disguised as several targets; a failed-only retry never
   repeats a succeeded target; a duplicate target never double-writes.
8. **Missing post-lock authority.** Re-read every mutable state after the lock. Order:
   `lock → current membership/session → assignment → resource → proposal state`. Avoid
   lock inversion, unordered multi-subject locks, a preview standing in for confirm
   authority, and a worker trusting a stale author snapshot.
9. **A permission exemption without replay redaction.** Adding one means auditing every
   transcript, event, message, receipt, error response and cross-tenant read. Lost
   assignment, role demotion, an inactive subject, or a deleted resource must fail the
   whole sensitive branch closed.
10. **Process-local queue recovery.** Correctness may not rest on an in-memory listener,
    a failure callback, the current process, or a best-effort async callback. Use durable
    proof, a generation fence, a deterministic job id, an outbox generation,
    stale-dispatch redrive, a crash/restart test, exact queue-rejection proof, and inert
    late events.
11. **Incomplete migrations.** A schema-file edit is not a migration. Check: enum
    predecessor migration, forward constraint migration, constraint, trigger, policy,
    function ownership, execute grants, runtime allowlist, startup readiness, retention
    clearing, private-column projection exclusion, a from-zero run, and the policy
    re-apply. Never verify only against a shared database someone hand-patched.
12. **Missing UI parity.** A procedure is not a product flow. See §6.
13. **Missing no-op / idempotency / retry coverage.** No-op, exact retry, double confirm,
    concurrent confirm, stale preview, rollback, post-lock drift, notification or queue
    failure, immutable receipt, zero duplicate audit rows, zero duplicate external effect.
14. **A surviving canonical-service bypass.** If a new service claims to be the only
    product boundary, no raw writer, convenience wrapper, compatibility export, alternate
    route orchestration or direct low-level mutation path may remain public.

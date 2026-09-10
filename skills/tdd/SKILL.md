---
name: tdd
description: Test-driven development. Use when the user wants to build features or fix bugs test-first, mentions "red-green-refactor", or wants integration tests.
---

# Test-Driven Development

TDD uses a failing behavior test to drive one small implementation at a time.
Read the relevant guidance once and apply it throughout the loop; do not reload
unchanged skill references for every test.

When exploring the codebase, read `CONTEXT.md` (if it exists) so test names and interface vocabulary match the project's domain language, and respect ADRs in the area you're touching.

## What a good test is

Tests verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't. A good test reads like a specification — "user can checkout with valid cart" tells you exactly what capability exists — and survives refactors because it doesn't care about internal structure.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Seams — where tests go

A **seam** is the public boundary you test at: the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

**Test at agreed seams.** Seams named by an approved ticket, acceptance contract,
project policy or earlier user answer are already authorized. Record and use them
without asking again. Choose routine test organization from the existing public
interfaces; ask only when choosing a new boundary requires an unresolved product
decision or materially changes the agreed scope.

## Anti-patterns

- **Implementation-coupled** — mocks internal collaborators, tests private methods, or inspects storage details unrelated to the contract. The tell: the test breaks when you refactor but behavior hasn't changed. Database assertions are appropriate when persistence, authorization, rollback or absence of writes is itself the agreed observable contract; drive the actual command first.
- **Tautological** — the assertion recomputes the expected value the way the code does (`expect(add(a, b)).toBe(a + b)`, a snapshot derived by hand the same way, a constant asserted equal to itself), so it passes by construction and can never disagree with the code. Expected values must come from an independent source of truth — a known-good literal, a worked example, the spec.
- **Horizontal slicing** — writing all tests first, then all implementation. Bulk tests verify _imagined_ behavior: you test the _shape_ of things rather than user-facing behavior, the tests go insensitive to real changes, and you commit to test structure before understanding the implementation. Work in **vertical slices** instead — one test → one implementation → repeat, each test a **tracer bullet** that responds to what the last cycle taught you.

## Rules of the loop

- **Red before green.** Write the failing test first, then only enough code to pass it. Don't anticipate future tests or add speculative features.
- **One slice at a time.** One seam, one test, one minimal implementation per cycle.
- **Keep cleanup bounded.** After green, make necessary local refactors while preserving
  behavior and rerun affected tests when inputs change. Follow the active review policy;
  `review=none` does not require a later Code Review/Evaluator stage.

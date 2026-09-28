# UI acceptance for the current ticket

Read this only when a ticket changes what a user sees or how they interact with the
product. A backend change can qualify if it changes a displayed state or interaction;
a frontend file change with no user-visible effect does not automatically qualify.
This stage supplements implementation and default code review. It is required for UI
tickets even with `review=off`, and remains separate from repository static hooks and CI.

## Before implementation

Use the available `ui-ux-pro-max` skill to assess the affected interface against the
existing approved design, copy, components and interaction patterns. Read only guidance
relevant to the change. Preserve approved wording and established UI; recommendations do
not authorize a redesign. If the requested solution needs a substantial unapproved
layout or interaction change, establish the concrete proposal and obtain the approval
required by the user's or project's rules before implementing that change.

Define a focused acceptance plan from this ticket's criteria and actual UI impact:

- The affected pages/components and the specific actions or journeys that change.
- The affected states, such as validation, loading, empty, error or success, only where
  relevant to those changes.
- The viewport sizes needed to prove the affected layout or interaction, using project
  conventions instead of inventing an exhaustive device matrix.
- The approved appearance/behavior to compare against and the screenshots needed to
  inspect it. Capture the current local UI before changing it when a visual comparison
  is necessary and no valid baseline already exists.

Resolve browser commands, local server, fixtures, authentication and test authorization
from the project. Standing project authorization is sufficient within its stated scope;
this skill supplies no additional permission for environments, data changes or tests.
Record missing tooling or access early and prepare what existing authorization permits.
If `ui-ux-pro-max` is unavailable, record the missing capability instead of silently
claiming its assessment happened.

## After a stable implementation

Coordinate independent code review, when enabled, with focused UI/UX review and actual
Playwright acceptance. The code reviewer examines the current ticket's diff without
rerunning tests. The orchestrator or a UI reviewer assesses the interface and evidence
against the approved design and ticket criteria; the implementation worker handles fixes.
These responsibilities may share one review pass, but code inspection alone cannot prove
browser behavior or visual correctness.

Use the project's supported Playwright runner or Playwright browser tooling against the
running local candidate. Exercise the scoped actions and states, verify their observable
results, and capture the necessary screenshots. Actually open and inspect the relevant
screenshots for layout, readability, feedback and consistency with the approved design.
Saving image files or receiving a passing test exit code does not constitute visual
inspection. Do not substitute mockups or static source inspection for the running UI.

Reuse valid Playwright results and screenshots produced during implementation when they
cover these criteria and still match the relevant code, configuration, fixtures and
viewports. Inspect those images rather than running the same browser flow again solely
for handoff. Do not turn this step into site-wide browsing or a full regression suite.

Keep a compact external receipt identifying the candidate, scoped page/flow/state and
viewport, actual assertions/results, screenshot locations and inspection outcome, and
relevant input fingerprints. Store evidence in the project's authorized local location;
avoid capturing or publishing secrets or real customer content. Never claim that a
missing, skipped, stale or uninspected result passed.

## Preserve acceptance through final integration

Before any operation that can remotely merge a pool containing UI tickets, record the
accepted candidate identity after all implementation, note and commit changes. Confirm
that all required UI receipts still apply to the final candidate's relevant inputs,
including changes made by later tickets. A passed static hook does not establish UI
acceptance. Reuse valid browser evidence; a note-only or otherwise irrelevant input
change does not require rerunning unchanged UI checks.

Use the repository's supported accepted-candidate/tree fence, or an equivalent
prepare-then-merge workflow that rejects any helper-created or selected candidate whose
identity differs from the accepted one before invoking remote merge. The repository owns
its commands and options. The boundary must stop a helper that integrates a newer target
from authorizing remote merge of that changed candidate before its affected acceptance
is established. Checking the head only before calling an unfenced all-in-one helper is
insufficient. Preserve repository freshness protections and confirm the actual merge
result and required audit; this fence does not make concurrent cross-clone updates atomic.

When integration stops at this boundary, preserve the same branch and PR. Reconcile the
changed code and note, finish necessary commits through the repository hook, and inspect
which required UI evidence was invalidated. Run only the affected authorized Playwright
checks and inspect their new screenshots. Retain unchanged evidence with the reason it
still applies. Only after required acceptance is established for that resulting candidate
record its new accepted identity, update the same PR and retry the guarded merge. Never
advance the fence to the latest head merely to make the helper proceed.

If the repository has no preparation boundary or identity fence that stops a changed
helper candidate before remote-merge invocation, leave final integration blocked and
report the missing capability. Do not merge with stale UI evidence or treat a clean
automatic merge as UI acceptance.

## Repairs and blockers

A behavior failure, visual defect or missing required acceptance stays on the current
ticket and returns to its same worker. Fix the demonstrated problem within scope, then
repeat only failed or invalidated assertions and screenshot inspection. Retain evidence
whose relevant inputs are unchanged; do not repeat the independent code-review pass or
all earlier ticket checks merely because one UI check failed.

If tooling, the server, authentication or execution authorization prevents required UI
acceptance, record the exact blocker and keep the ticket `validating` or `blocked`.
Local work or intermediate checked commits may be preserved, but do not mark the ticket
`committed`, advance the queue or count code review as a substitute. Resume the same
worker and branch when the blocker clears. A later integration change that invalidates
UI evidence requires only the affected authorized checks before final delivery.

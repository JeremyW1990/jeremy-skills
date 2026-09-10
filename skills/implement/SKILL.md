---
name: implement
description: "Implement a piece of work based on a spec or set of tickets."
---

Implement the work described by the user in the spec or tickets.

Use the `tdd` skill where applicable, at the approved ticket's seams. Those seams are
already agreed; do not ask the user to confirm them again. Reuse the orchestrator's
current context/setup and read the relevant source, rather than rediscovering the pool
or repeating dependency installation and base synchronization.

Follow the validation and review policy passed by the user, active project, or ticket
orchestrator. Use focused tests and affected typechecks for development feedback.
Meet every ticket acceptance criterion, including relevant database and UI behavior.
When an orchestrator or project command owns final verification, report the focused
results and let that final gate run the remaining required checks once. Otherwise run
the project's required final suite once. Do not duplicate unchanged checks for handoff.

Inspect the resulting diff for unintended changes. By default, use /code-review once
done; when the user or active project selects `review=none`, skip the separate review
and Evaluator stage without dropping acceptance tests.

Commit your work to the current branch.

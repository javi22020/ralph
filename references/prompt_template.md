# PROMPT.md template

Copy this to `.ralph/PROMPT.md` at the start of a Ralph run and fill it in with the user (or infer reasonable answers and confirm).

```markdown
# <Project / task name>

## Goal
<One paragraph. What does success look like in plain English? A reader who has never seen this project should understand what we are trying to achieve.>

## Context
<2–5 bullets on the relevant background: which codebase, which language/framework, who the consumer is, why this matters now.>

## Hard constraints
- <Language / framework version locked, e.g. "Python 3.11, no new dependencies">
- <Files or directories that are off-limits, e.g. "do not modify tests/">
- <Style / convention requirements, e.g. "follow existing module layout">
- <Performance, security, or compatibility constraints>

## Out of scope
- <Things that look related but should not be done in this run>
- <Refactors that are tempting but not the goal>

## Exit criteria
<This is the most important section. Each criterion must be objectively checkable.>

- [ ] <e.g. `pytest tests/` exits 0>
- [ ] <e.g. `npm run build` succeeds with no warnings>
- [ ] <e.g. every checkbox in fix_plan.md is checked>
- [ ] <e.g. the new endpoint returns 200 for the happy path and 4xx for the documented error cases>

## Notes for the loop
<Optional: anything the loop should know that does not fit above. Known flaky tests to ignore, environment quirks, links to relevant docs, etc.>
```

## Tips for writing a good PROMPT.md

- **Be ruthless about exit criteria.** "Code is clean and well-tested" is not a criterion; "all tests pass and coverage is ≥80% on touched files" is. The dual-condition exit gate is only as good as these criteria.
- **Constraints earn their keep by being terse and specific.** Vague constraints ("write idiomatic code") get ignored or cause arguments; specific ones ("use the existing `db.py` connection pool, do not introduce a new one") get followed.
- **Out-of-scope is half the prompt.** Most of the value of `PROMPT.md` is in keeping the loop from drifting. List the tempting-but-wrong rabbit holes explicitly.
- **One PROMPT.md per goal.** If the user changes the goal mid-run, stop the loop, edit `PROMPT.md`, reset `state.json`, and start a new run. Do not silently mutate the goal.

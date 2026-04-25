# fix_plan.md template

Copy this to `.ralph/fix_plan.md` at the start of a Ralph run. The loop reads and edits this file every iteration.

```markdown
# Fix plan

Last updated: <iteration N, ISO timestamp>

## Conventions

- `[ ]` open task, ready to start
- `[~]` partial progress (some work done, more remains — add a sub-bullet explaining state)
- `[x]` done and verified (verification command + result must be in the loop log)
- `[!]` blocked (sub-bullet explains the blocker)
- `[-]` cancelled / out of scope (sub-bullet explains why)

Tasks are ordered top-to-bottom by intended execution order. The loop always picks the first `[ ]` it finds.

## Tasks

### Phase 1 — <e.g. "stand up the structure">
- [ ] <task 1: a single deliverable, ideally one iteration of work>
- [ ] <task 2>
- [ ] <task 3>

### Phase 2 — <e.g. "make tests pass">
- [ ] <task 4>
- [ ] <task 5>

### Phase 3 — <e.g. "polish and verify exit criteria">
- [ ] <task 6>
- [ ] Run full verification suite and confirm every exit criterion in PROMPT.md
```

## Tips for writing a good fix plan

- **Each task should be one iteration.** If a task feels like "and then a few more things", split it. Tasks that span multiple iterations are how loops get stuck.
- **Order by dependency, then by signal.** Risky / blocking work first; cosmetic work last. Within a phase, prefer tasks that produce a fast verifiable signal (a green test) over tasks that don't (renaming a variable).
- **Append, don't rewrite.** When the loop discovers new sub-problems, append them under the relevant phase. Do not silently delete tasks — mark them `[-]` with a reason if they become out of scope.
- **The last task is always "verify exit criteria".** This forces a final pass that maps the work back to `PROMPT.md` rather than declaring victory on vibes.

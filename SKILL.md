---
name: ralph
description: Run an autonomous iterate-test-replan loop until a multi-step coding task is complete. Use this whenever the user asks Claude Code to work autonomously through a backlog, fix plan, or specification with minimal supervision — phrases like "ralph this", "loop until done", "keep iterating until tests pass", "work autonomously on this", "finish the fix plan", "run until the build is green", or any request that implies sustained unattended progress across many turns. Prefer this skill over ad-hoc "just keep working" loops because it adds explicit exit criteria, a circuit breaker against stuck iterations, subagent isolation for tests and review, and persistent state that survives session compaction.
---

# Ralph: Autonomous Iterate–Test–Replan Loop

## What this is

Ralph is an autonomous development loop adapted for Claude Code. The pattern was originally a shell script that re-invoked the Claude Code CLI in a loop until a project was finished; this skill brings the same idea *inside* a single Claude Code session, leveraging native subagents and the Task tool instead of process-level re-invocation.

The loop has four phases per iteration: **plan → implement → test → evaluate**. After evaluation, Claude either records progress and continues, or replans, or exits. Termination requires *both* a heuristic completion signal *and* an explicit `EXIT_SIGNAL: true` from Claude — this dual-condition gate is what keeps the loop from quitting on a phrase like "this part looks done" while real work remains.

## When to use this skill

- "Ralph this", "loop on this", "keep iterating until X"
- The user gives a fix plan, backlog, spec, or PRD and wants it executed end-to-end
- The user wants Claude to work autonomously across many turns without check-ins
- A test suite is failing in many places and the user wants Claude to grind through fixes
- A feature has clear acceptance criteria and the user wants it built and verified

## When NOT to use this skill

- Single-shot questions or one-file edits — just do them directly
- Tasks that require human judgment at each step (design decisions, UX trade-offs, security review of unknown code, prod deploys)
- Open-ended exploration where there is no falsifiable exit criterion
- The user is paired with Claude in real time and wants conversational back-and-forth

## Setup phase (run exactly once at the start)

Before entering the loop, do all of the following:

1. **Confirm the goal in writing.** If the user did not provide a `PROMPT.md`, write a short one in `.ralph/PROMPT.md` capturing: the goal, hard constraints (libraries, language, deadlines), out-of-scope items, and what "done" looks like. See `references/prompt_template.md`.

2. **Build a fix plan.** Decompose the goal into a checklist in `.ralph/fix_plan.md`. Keep tasks small (each one ideally completable in a single iteration). Order by dependency, then by risk (do risky/blocking work first). See `references/fix_plan_template.md`.

3. **Define exit criteria explicitly.** Write them at the top of `.ralph/PROMPT.md` under `## Exit criteria`. Good criteria are objective: "all tests in `tests/` pass", "`npm run build` succeeds with no warnings", "every checkbox in fix_plan.md is checked". Avoid "looks good" or "is reasonably complete".

4. **Create `.ralph/state.json`** with the initial loop state:
   ```json
   {
     "iteration": 0,
     "max_iterations": 50,
     "completion_indicators": 0,
     "consecutive_no_progress": 0,
     "last_error_signature": null,
     "same_error_streak": 0,
     "circuit_breaker": "CLOSED"
   }
   ```

5. **Use the TodoWrite tool** to mirror the top 5–10 tasks from the fix plan into the in-session todo list. Update todos as work progresses; this is how the user sees status without having to read files.

6. **Announce the plan.** Show the user the goal, the fix plan, and the exit criteria, then proceed.

## The main loop

For each iteration `i = 1..max_iterations`:

### 1. Plan this iteration

Pick the *next unchecked* task from `fix_plan.md`. If multiple tasks are unblocked, prefer the one that produces the fastest verifiable signal (a failing test you can drive green is more valuable than refactoring docs).

Read just enough surrounding code to do the task — do not try to hold the whole codebase in context. For larger explorations, delegate to a subagent (see "Subagent delegation" below).

### 2. Implement

Make the change. Keep the diff focused on the task; resist scope creep. If you discover a new sub-problem, append it to `fix_plan.md` rather than handling it inline.

### 3. Test

Run the relevant verification — unit tests, type-check, lint, build, or whatever the project defines. Two patterns:

- **Short check (under ~30 seconds)**: run inline with the Bash tool. Read the output directly.
- **Long check (tests, builds, integration runs)**: spawn the **ralph-tester subagent** (see `agents/ralph-tester.md`). The subagent runs the suite, parses output, and returns a structured summary — failing test names, error signatures, and a one-line verdict — without dumping thousands of lines into the main context. For genuinely long-running suites, the subagent can launch the command with `run_in_background: true` and poll output.

If verification cannot be run (no tests defined yet, or the change is in a layer without coverage), say so explicitly in the iteration log instead of declaring success silently.

### 4. Evaluate

Decide what just happened. Three outcomes:

- **Progress** — the task is complete and verified. Check it off in `fix_plan.md`, increment iteration, continue.
- **Partial progress** — change works but reveals new work. Append the new work to `fix_plan.md`, mark partial progress in the log, continue.
- **No progress / regression** — verification failed in the same way as before, or a different change broke something. Update `same_error_streak` and `consecutive_no_progress` in `state.json`, then check the circuit breaker (next section).

For non-trivial decisions (architectural choice, ambiguous failure, change introduces risk in unrelated code), spawn the **ralph-reviewer subagent** (see `agents/ralph-reviewer.md`) to get a fresh-context critique on the diff before continuing.

### 5. Record state and emit the status block

At the end of every iteration, append a `RALPH_STATUS` block to `.ralph/loop.log`:

```
=== Iteration <N> ===
TASK: <one-line description of what was attempted>
RESULT: <progress | partial | no_progress | regression>
VERIFICATION: <how it was checked, e.g. "ran pytest tests/test_auth.py — 12/12 passed">
NOTES: <anything notable: new tasks added, decisions made, files touched>

RALPH_STATUS:
  completion_indicators: <integer ≥ 0>
  EXIT_SIGNAL: <true | false>
  reason: <free-text justification>
```

`completion_indicators` is your honest count of "things that suggest we're done": all checkboxes ticked, all tests green, build passing, exit criteria from PROMPT.md objectively met. `EXIT_SIGNAL` is the explicit yes/no on whether you actually believe the goal is achieved end-to-end.

### 6. Check the exit gate

**Exit only if BOTH of these hold:**
- `completion_indicators >= 2`, AND
- `EXIT_SIGNAL: true`

In every other case, continue. In particular: high `completion_indicators` with `EXIT_SIGNAL: false` means *keep working* — the surface signals look good but the model knows there is more to do. Trust the explicit signal over the heuristic.

### 7. Check the circuit breaker

Open the breaker (stop the loop, surface to the user) if any of these hold:

- `iteration >= max_iterations` (default 50)
- `consecutive_no_progress >= 3` (three iterations in a row produced no checked task and no test transition green)
- `same_error_streak >= 5` (the same error signature has appeared five iterations in a row)
- A destructive or irreversible action is needed and has not been pre-authorised by the user (e.g. `rm -rf` outside the working tree, force-push, schema migration on a real database)

When the breaker opens, **stop, do not silently retry**. Spawn the **ralph-replanner subagent** (see `agents/ralph-replanner.md`) with the recent log slice and current `fix_plan.md`. It returns one of: a revised fix plan, a recommendation to ask the user a specific question, or a recommendation to declare the task infeasible and explain why. Surface the replanner's recommendation to the user and wait for direction.

## Subagent delegation

Spawn subagents via the Task tool for any of these. Subagents run in their own context window and only return their final message — this is the main mechanism for keeping the parent context clean across a long run.

| When | Subagent | What it does |
|---|---|---|
| Need to explore a large unfamiliar codebase before implementing | built-in `Explore` | Read-only, fast codebase search |
| Long test runs, build runs, integration suites | `ralph-tester` (see `agents/ralph-tester.md`) | Runs verification, returns structured pass/fail summary |
| Risky diff, architectural choice, or "is this change actually correct?" | `ralph-reviewer` (see `agents/ralph-reviewer.md`) | Fresh-context critique on the diff and surrounding code |
| Circuit breaker opened, or fix plan looks wrong | `ralph-replanner` (see `agents/ralph-replanner.md`) | Reads recent log + plan, proposes revised plan or escalates |
| Genuinely independent parallelisable work (e.g. add docs to ten files, refactor ten modules with the same pattern) | `general-purpose` × N in parallel | Same turn, multiple Task calls; aggregate results |

Two operational rules:

1. **Pass everything the subagent needs in the prompt string.** The subagent starts with no parent context. Include the file paths, the failing test names, the relevant error messages, the exit criteria — everything. A vague prompt produces a vague answer.
2. **Do not nest deep agent trees.** A subagent spawning another subagent is allowed but rarely a good idea inside Ralph; the loop is already the orchestration layer.

The named subagent files in `agents/` are templates. If the user has installed them in `.claude/agents/`, invoke them by name (`subagent_type: ralph-tester`). If not, use `subagent_type: general-purpose` and paste the agent body into the prompt — the result is equivalent, just less ergonomic.

## State files

All loop state lives under `.ralph/` in the project root. Treat this directory as the source of truth that survives session compaction.

```
.ralph/
├── PROMPT.md       # goal, constraints, exit criteria — written once at setup, edited rarely
├── fix_plan.md     # ordered checklist — edited every iteration
├── state.json      # iteration counters, circuit breaker state — edited every iteration
└── loop.log        # append-only log of RALPH_STATUS blocks — append every iteration
```

If the session compacts mid-loop, the next turn must re-read `state.json`, `fix_plan.md`, and the tail of `loop.log` before continuing. Do this *before* starting iteration N+1.

## Exit and final report

When the dual-condition exit gate fires, do not stop silently. Produce a final report containing:

1. The original goal (one paragraph from `PROMPT.md`).
2. Each exit criterion and how it was verified (with the actual command output or test count).
3. A diff summary: files added/modified/deleted.
4. Any items that were *not* completed and why (often there will be 0 — but if you punted on something, say so).
5. Suggested next steps if the user wants to continue beyond the original goal.

Then update the in-session todos to reflect completion and stop.

## Safety guardrails

- **Never bypass the user's permission rules.** If a tool call is denied, surface that to the user — do not retry indefinitely or attempt to work around it. The circuit breaker should open after the second denial of the same operation.
- **Do not make the loop run "headlessly" past prompts.** If something genuinely needs user input (a credential, a design choice, an irreversible action), stop and ask. Ralph is autonomous, not unilateral.
- **Be honest in `RALPH_STATUS`.** The dual-condition gate only works if `EXIT_SIGNAL` reflects genuine belief. If the work is partly done, set `false` and continue. Optimistic exits are worse than long loops.
- **Respect cost.** Long autonomous loops consume tokens and API budget. If `max_iterations` is approaching with substantial work remaining, open the breaker early and ask the user whether to continue.

## Example invocation

User: *"Read the failing tests in `tests/`, then ralph this until everything passes."*

Claude (this skill is now active):
1. Setup: read failing tests, write `.ralph/PROMPT.md` ("Goal: get all tests in `tests/` passing without changing test files"), build `.ralph/fix_plan.md` from the failures, define exit criterion ("`pytest tests/` exits 0"), initialise `state.json`, mirror top tasks into todos, announce plan.
2. Loop iteration 1: pick first failing test → read failing module → implement fix → spawn `ralph-tester` to confirm just that test now passes and nothing else regressed → record `RALPH_STATUS` → continue.
3. Loop iteration 2..N: same shape, with periodic `ralph-reviewer` calls on larger diffs.
4. When all checkboxes are ticked AND `pytest tests/` exits 0 AND `EXIT_SIGNAL: true`: produce final report.

## Reference files

- `references/prompt_template.md` — fill-in template for `.ralph/PROMPT.md`
- `references/fix_plan_template.md` — fill-in template for `.ralph/fix_plan.md`
- `agents/ralph-tester.md` — subagent definition for verification runs
- `agents/ralph-reviewer.md` — subagent definition for diff review
- `agents/ralph-replanner.md` — subagent definition for circuit-breaker recovery

The `agents/` files are valid Claude Code subagent definitions; copy them into `.claude/agents/` to make them invokable by name. If they are not installed, use them as prompt templates with `subagent_type: general-purpose`.

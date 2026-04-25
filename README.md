# Ralph for Claude Code (skill)

A Claude Code skill that turns an autonomous iterate–test–replan loop into a first-class capability of a single Claude Code session. Inspired by [Geoffrey Huntley's Ralph technique](https://ghuntley.com/ralph/) and the [ralph-claude-code](https://github.com/frankbria/ralph-claude-code) shell-script implementation, but built around Claude Code's native subagents instead of CLI re-invocation.

## What it does

When the user asks Claude Code to "ralph this", "loop until done", or anything similar, this skill activates and runs a structured loop:

1. **Setup** — captures the goal, builds a fix plan, defines objective exit criteria, initialises persistent state under `.ralph/`.
2. **Loop** — for each iteration: pick the next task, implement, verify (delegating to the `ralph-tester` subagent for long runs), evaluate, log, decide whether to continue.
3. **Exit** — only when *both* heuristic completion signals AND an explicit `EXIT_SIGNAL: true` agree. Optimistic exits are deliberately hard.
4. **Recover** — when stuck (same error repeated, no progress, max iterations), the `ralph-replanner` subagent triages and proposes either a revised plan, a specific question for the user, or an infeasibility declaration.

## Installation

### As a Claude Code skill (recommended)

Copy the entire `ralph/` folder into either:
- `.claude/skills/ralph/` (project-level — only available in this repo)
- `~/.claude/skills/ralph/` (user-level — available everywhere)

Claude Code will discover the skill on its next invocation.

### Bundled subagents

The three named subagents under `agents/` are valid Claude Code subagent definitions. To make them invokable by name (`subagent_type: ralph-tester`, etc.), copy them into `.claude/agents/` or `~/.claude/agents/`:

```bash
cp ralph/agents/*.md ~/.claude/agents/
```

If you skip this step, the skill still works — the main agent will use `subagent_type: general-purpose` and inline the agent body. The result is functionally identical, just slightly more verbose at invocation time.

## How to invoke

You don't, directly. Anything that sounds like "keep working on this autonomously" should trigger it:

- *"Ralph this until tests pass."*
- *"Read TODO.md and loop until everything's checked off."*
- *"Iterate on this until the build is green."*
- *"Work autonomously through the fix plan in `.ralph/fix_plan.md`."*

The skill does its own setup if `.ralph/` doesn't exist; otherwise it picks up where the previous run left off.

## File layout

```
ralph/
├── SKILL.md                             # The skill itself — read this for the full design
├── README.md                            # You are here
├── references/
│   ├── prompt_template.md               # Template for .ralph/PROMPT.md
│   └── fix_plan_template.md             # Template for .ralph/fix_plan.md
└── agents/
    ├── ralph-tester.md                  # Subagent: runs verification, returns structured summary
    ├── ralph-reviewer.md                # Subagent: critiques diffs in fresh context
    └── ralph-replanner.md               # Subagent: triages stuck loops, proposes recovery
```

## Design notes

A few choices that are worth flagging.

**Dual-condition exit gate.** The single most important property of Ralph is that it does not exit on a vibe. The loop terminates only when (a) at least two heuristic completion signals are present *and* (b) the model explicitly emits `EXIT_SIGNAL: true`. Either alone is insufficient. This costs some extra iterations near the end of a run; in exchange, the loop almost never declares victory while real work remains.

**Subagent-first verification.** Test runs, builds, and integration suites go through the `ralph-tester` subagent rather than running inline. This is not about parallelism — it is about *context hygiene*. A 4000-line pytest output dumped into the parent context is 4000 lines the parent has to reason past for the rest of the run. The tester returns a 30-line structured summary instead.

**Persistent state survives compaction.** Everything important lives in `.ralph/` on disk: goal, plan, iteration counters, log. If the session compacts mid-loop, the next turn re-reads the files and continues. This is also how a Ralph run can be paused and resumed across sessions.

**Replanner as the only escape valve.** The parent agent is *not* allowed to silently abandon a stuck plan or silently expand the goal. The only sanctioned response to "stuck" is to invoke the replanner, which makes one of three explicit recommendations. This is what keeps autonomy from drifting into chaos.

**Tasks API: what's used and what isn't.** Three different things get called "Tasks API" in Claude Code's surface area:

- *The Task tool* (subagent spawning) — used heavily. This is the primitive the whole skill is built on.
- *Background tasks* (`run_in_background: true` on Bash calls) — used optionally by the tester subagent for long-running verification commands.
- *Scheduled routines / cloud Tasks API* — **not used.** Those are time-triggered cron jobs that run on Anthropic's infrastructure; Ralph is a task-triggered, in-session loop. Mixing them would muddle the model.

## Limits and caveats

- **Not for irreversible production actions.** Ralph is autonomous, not unilateral. Anything destructive (force-push, prod migrations, `rm -rf` outside the working tree) opens the circuit breaker by design and surfaces to the user.
- **Cost is real.** A long Ralph run consumes tokens and API budget. The default `max_iterations` of 50 is conservative; raise it deliberately.
- **Exit criteria are everything.** A weak `PROMPT.md` produces a weak loop. The fastest way to make Ralph misbehave is to give it vague success conditions and walk away.

## License

MIT-style: do whatever you want with it. No warranty.

---
name: implement-task
description: Implement one roadmap task T-xxx of roadmap/roadmap.md by following its file roadmap/T-xxx.md, after verifying that every dependency (tasks DONE, arbitrages SETTLED, external dependencies PROVIDED). Runs the acceptance checks, then updates the task status in roadmap.md and T-xxx.md (DONE, BLOCKED…) and creates new arbitrages (arbitrage.md + ARB-xxx.md) when the work raises a decision. Use when asked to implement, execute or work on a roadmap task.
argument-hint: "T-xxx (e.g. T-007)"
disable-model-invocation: true
---

# Implement task

Implements **exactly one** task of the roadmap: `$ARGUMENTS`.

Files (workspace root): `roadmap/roadmap.md` (task table §5 = authoritative task status; external dependencies §6), `roadmap/T-xxx.md` (the task), `roadmap/arbitrage.md` (synchronised copy of the arbitrages), `roadmap/ARB-xxx.md` (the `Status` field of its header table is authoritative). Read `roadmap/roadmap.md` §2 (readiness rule, status transitions, rules for every task) before anything else.

## Step 0 — Parse the argument

- Accept `T-7`, `T-007`, `t-007` or `7`; normalise to `T-` + three digits.
- No argument, or no row with that number in the table of `roadmap.md` §5, or no file `roadmap/T-xxx.md`: answer that the task does not exist and **stop**.

## Step 1 — Gate: verify every dependency BEFORE doing anything else

No code, no file, no status may be changed before this step is complete.

1. Read the row of the task in `roadmap.md` §5 and the `Dependencies` row of the header table of `T-xxx.md`. If the two lists differ, use their **union** and report the drift.
2. Current status of the task:
   - `DONE` → answer that the task is already done (quote its completion record) and **stop**.
   - `WIP` → answer that the task is already in progress and **stop** (do not take over silently; the user may set it back to `TODO` explicitly).
   - `BLOCKED` → read the *Blocking problems* section of `T-xxx.md` if it exists. A problem recorded there that is not marked resolved → answer with it and **stop**. Otherwise continue (the block came from dependencies, re-checked below).
   - `TODO` → continue.
3. Check each dependency at its source of truth, never from memory:

   | Dependency | Satisfied only when | Where to read |
   |---|---|---|
   | `T-xxx` | status `DONE` | *Status* column of `roadmap.md` §5 |
   | `ARB-xxx` | `Status` = `SETTLED` | header table of `roadmap/ARB-xxx.md` (if the `arbitrage.md` row disagrees, the ARB file wins; report the drift) |
   | `EXT-xxx` | status `PROVIDED` | table of `roadmap.md` §6 |

4. **If at least one dependency is not satisfied, do not implement anything.** Answer:
   - that the skill cannot be executed for this task;
   - a table of every unsatisfied dependency: id, title/subject, current status, what is needed (`T`: implement it — say whether it is itself ready, and if not, list its own unsatisfied dependencies, recursively, without duplicates; `ARB`: settle it with `/settle-arbitrage ARB-xxx`, with the questions listed in its file; `EXT`: who provides it, from §6);
   - the tasks that **are** ready now (status `TODO` and every dependency satisfied), as a suggestion.
   
   The only file change allowed in this case: if the task is `TODO` and an unsatisfied dependency is an `ARB` or an `EXT`, set it to `BLOCKED` (roadmap row + `T-xxx.md` Status row, see Step 6). Then **stop**.

## Step 2 — Start

1. Set the task to `WIP` in the roadmap table (change only that cell) and in the `T-xxx.md` Status row.
2. Read `T-xxx.md` entirely, then everything listed in its *Context (read first)*: the referenced sections of `architecture-plan.md`, `contracts/openapi.yaml`, `klesia_api/docs/specs/SPECIFICATIONS.md`, `klesia_webapp/docs/specs/SPECIFICATIONS.md`, and the related arbitrages (read the *Decision* section of every settled one: a decision overrides the default written in the task file).
3. Read the `CLAUDE.md` of each application the task touches (`klesia_api/CLAUDE.md`, `klesia_webapp/CLAUDE.md`) and the root `CLAUDE.md`, and use the agents and skills named in *Suggested agent / skill* when they help.
4. Inspect the existing code the task builds on before changing it.

## Step 3 — Implement

- Follow the *Plan of actions* in order. Stay inside *Scope*: anything outside it becomes a new task (Step 5), never an unrequested change.
- Respect the rules of `roadmap.md` §2.3: contract first, no speculative capability, per-application rules, no secret in any tracked file, log or report.
- **Contract-first stop.** If the task needs a change to anything observable on the HTTP boundary (route, schema, status code, header, auth rule, CORS) that the contract and the owning specification do not already describe, do not change code to make it work: create an arbitrage (Step 5) and set the task `BLOCKED`.
- **Decision stop.** If the plan cannot be executed without a choice that no specification, contract or settled arbitrage makes (missing value, contradictory sources, a new dependency or package not named by the task, a security trade-off), do not guess: create an arbitrage (Step 5). If the rest of the task can still be finished without that decision, finish it and add the arbitrage as a related (non-gating) item; otherwise set the task `BLOCKED`.
- Never weaken a test, a lint rule, an analyser, authentication or a check to make the build pass.
- Do not commit, push or deploy unless the user asked for it; `klesia_api/` and `klesia_webapp/` are separate Git repositories.

## Step 4 — Verify

1. Run every command of the task's *Verification* section, then the full build and test suites of each application touched:
   - `klesia_api`: `dotnet build Klesia.sln` and `dotnet test Klesia.sln` (plus the SQL project build when `database/` changed);
   - `klesia_webapp`: `npm.cmd run type-check`, `lint`, `format:check`, `test:run`, `build` (scripts that exist);
   - when the HTTP boundary or `src/models/demand.ts` was touched: the `check-contract` skill on the relevant steps.
2. Check every acceptance criterion one by one and tick it (`- [x]`) only with evidence (a command actually run, a test name, a file and line).
3. Environment-dependent checks (real Entra, Azure resources, disposable database) whose prerequisites are missing are reported as *not run* with the reason; they are never assumed to pass. A criterion that requires them cannot be ticked.
4. A failing test or check: fix it within scope and re-run. If it cannot be fixed without leaving the scope or taking a decision → Step 5.

## Step 5 — New arbitrages and follow-up tasks

**New arbitrage** (a decision or open question raised by the work):

1. Next number = highest `ARB-xxx` found in `roadmap/arbitrage.md` **and** in the `roadmap/ARB-*.md` file names, plus one, three digits.
2. Create `roadmap/ARB-xxx.md` with the template below (`Status` = `OPEN`). `Blocking` = `YES` if the current task (or another one) cannot be completed before the decision, else `NO` with the provisional default stated.
3. Append a row **at the end** of the table in `arbitrage.md` §3, copying the header fields exactly (`Concerning tasks` with backticked ids), and add a line to its change log.
4. If `Blocking` = `YES`: add the `ARB-xxx` to the `Dependencies` of every concerned task, in the roadmap row **and** in the `T-xxx.md` header. Otherwise add it to their *Related arbitrages (non-gating)*.

```markdown
# ARB-xxx — <Subject>

| Field | Value |
|---|---|
| Number | ARB-xxx |
| Subject | <Subject> |
| Description | <one sentence: what must be decided or answered> |
| Blocking | YES or NO |
| Concerning tasks | `T-xxx`, `T-yyy` |
| Status | OPEN |

## Context

<what was found, while implementing which task, with references to the specifications/contract/code>

## Questions

1. <question that must be answered to settle it>

## Options

| Option | Consequence |
|---|---|
| A. <option> (recommended) | <consequence> |
| B. <option> | <consequence> |

## Recommendation / provisional default

<recommended option; for Blocking = NO, the default the tasks apply meanwhile>

## Decision

*To be filled when settled: decision, date, decided by, follow-up tasks.*
```

**New follow-up task** (work discovered outside the scope, not a decision): next free `T-xxx` number, row appended **at the end** of the table in `roadmap.md` §5 with its dependencies, priority and area, and a `roadmap/T-xxx.md` file with the same structure as the existing task files (header table, gate, Goal, Context, Scope, Plan of actions, Acceptance criteria, Verification, Completion record). Its status is `BLOCKED` if it depends on an open `ARB`/pending `EXT`, else `TODO`. Add a line to the roadmap change log (§9).

## Step 6 — Update the status

Decide the final status:

| Outcome | Status |
|---|---|
| Every acceptance criterion ticked with evidence, all checks of Step 4 green | `DONE` |
| A gating arbitrage was created, an `EXT` is missing, or a problem cannot be solved within scope | `BLOCKED` |

Never leave the task `WIP` at the end of the skill.

Write it in both places:

1. `roadmap.md` §5: the *Status* cell of the task row (nothing else in the row, except added dependencies from Step 5).
2. `T-xxx.md`: the header row `| Status | ... |` becomes `| Status | <STATUS> (mirror of [roadmap.md](roadmap.md) §5) |`.

Then fill `T-xxx.md`:

- **DONE** → *Completion record*: date, changed files (or commits) per repository, commands run with their result, acceptance criteria evidence, items *not run* and why, arbitrages or tasks created.
- **BLOCKED** → add (or update) a section `## Blocking problems` just before *Completion record*: date, what blocks, the `ARB`/`EXT` ids, what was already done (files changed), and what remains. Keep the partial work only if it builds and its tests pass; otherwise revert it and say so.

When the task becomes `DONE`, list the tasks that are **now ready** (status `TODO`, every dependency satisfied); do not change their status.

## Step 7 — Report

Short report: final status; changes per repository; commands run and results; acceptance criteria met / not run; arbitrages and tasks created (with links); tasks now ready. Never claim a check passed unless it was run.

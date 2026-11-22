---
name: settle-arbitrage
description: Settle one roadmap arbitrage ARB-xxx by asking the user the questions needed to decide it, then recording the decision in roadmap/ARB-xxx.md, setting it SETTLED, synchronising roadmap/arbitrage.md and unblocking the tasks of roadmap/roadmap.md that were waiting for it. Use when asked to settle, decide, answer or close an arbitrage.
argument-hint: "ARB-xxx (e.g. ARB-004)"
disable-model-invocation: true
---

# Settle arbitrage

Settles **exactly one** arbitrage: `$ARGUMENTS`.

Files (workspace root): `roadmap/ARB-xxx.md` (its header table is authoritative: `Number`, `Subject`, `Description`, `Blocking`, `Concerning tasks`, `Status`), `roadmap/arbitrage.md` (synchronised copy, §3 table), `roadmap/roadmap.md` (task table §5, external dependencies §6, readiness rule §2).

This skill records a decision taken **by the user**. The agent prepares, asks, checks consistency and writes; it never decides on the user's behalf and never settles with an unanswered question.

## Step 0 — Parse the argument

- Accept `ARB-4`, `ARB-004`, `arb-004` or `4`; normalise to `ARB-` + three digits.
- No such file `roadmap/ARB-xxx.md` → answer that the arbitrage does not exist and **stop**.
- `Status` = `SETTLED` → show its *Decision* section and **stop**. Re-opening a settled arbitrage is done only on the user's explicit request, and is then recorded as a revision (date, reason) in the *Decision* section.

## Step 1 — Prepare

1. Read `ARB-xxx.md` entirely (Context, Questions, Options, Recommendation, any *Progress* section from a previous session).
2. Read what it references: the cited sections of `architecture-plan.md`, `contracts/openapi.yaml`, `klesia_api/docs/specs/SPECIFICATIONS.md`, `klesia_webapp/docs/specs/SPECIFICATIONS.md`, and the files `roadmap/T-xxx.md` of the concerning tasks, so that each question is precise and its consequences are known.
3. Build the question list: every numbered question of the *Questions* section that has no recorded answer yet, plus any question implied by the *Options* table (which option?). Keep each question short and answerable; split compound questions.
4. For each question prepare: the options (from the file, or the realistic ones), the **recommended** option marked as such with a one-line reason, and the consequence of each option for the concerning tasks.

## Step 2 — Ask the questions

- Use the interactive question tool of the host (e.g. `AskUserQuestion` in Claude Code, the ask-questions tool in VS Code); if none is available, ask in the chat and wait for the answer.
- Ask in small rounds (at most four questions per round), in the order of the file. Offer the prepared options, mark the recommended one, and always allow a free-form answer and an "I don't know yet / to be asked to <party>" answer.
- Start the first round with one sentence recalling the subject and why it matters (which tasks are blocked).
- **Never ask for a secret** (password, key, token, connection string with credentials). For a sensitive value, ask only *where it is stored* or *who provides it*; identifiers that are not secret but that the user considers sensitive are not written in the repository either (record "provided, stored in …").
- After each round, check the answers:
  - **Consistency with the sources.** An answer that contradicts a MUST of a specification, the contract or an architecture principle (`architecture-plan.md` §3) is explained to the user with the reference, and the user is asked to confirm. A confirmed contradiction is allowed but creates follow-up tasks that change the owning specification and the contract first (root `CLAUDE.md`, GOV-01).
  - **Consistency with other arbitrages.** Read the settled arbitrages that touch the same subject (e.g. ARB-009 environments, ARB-010 IaC ownership) and point out any conflict.
  - **Completeness.** Ask follow-up questions when an answer is ambiguous or opens a new question.
- Finally ask who takes the decision (name or role) if it was not given.

## Step 3 — Unanswered questions (partial session)

If at least one question remains unanswered (e.g. "to be asked to the DSI"):

1. Do **not** settle. Keep `Status` = `OPEN`.
2. Add or update a section `## Progress` just before *Decision* in `ARB-xxx.md`: date, each answer obtained so far, each question still open and who must answer it.
3. Report what is still missing and **stop**.

## Step 4 — Confirm, then write

1. Show the user a summary: the decision (one or two sentences), the answer to each question, the consequences, the follow-up tasks that will be created, and the tasks that will be unblocked. Ask for explicit confirmation. Without confirmation, nothing is written beyond the *Progress* section.
2. In `ARB-xxx.md`, replace the placeholder of the *Decision* section with:

```markdown
## Decision

| Field | Value |
|---|---|
| Decision | <one or two sentences> |
| Date | <YYYY-MM-DD> |
| Decided by | <name or role> |
| Follow-up tasks | `T-xxx`, … or — |

### Answers

1. <question> — <answer>

### Rationale and consequences

<why; impact on the concerning tasks; what differs from the provisional default, if anything>
```

   Remove the *Progress* section (its content is now in *Answers*).
3. Set `Status` = `SETTLED` in the header table. If follow-up tasks were created, add them to `Concerning tasks`.
4. In `arbitrage.md` §3, update the row of the arbitrage from the header (all fields, `Status` = `SETTLED`); add a line to its change log (§4).

## Step 5 — Follow-up tasks

Create a follow-up task when the decision requires work that no existing task covers, in particular:

- the decision differs from the provisional default the tasks applied (`Blocking` = `NO`) and already-written code or configuration must change;
- a specification, the contract or the architecture plan must be updated (contract-first sequence of `architecture-plan.md` §15.1);
- new infrastructure, a new package or a new procedure is decided.

For each: next free number (highest `T-xxx` in `roadmap.md` §5 and in the `roadmap/T-*.md` file names, plus one), a row appended **at the end** of the table in `roadmap.md` §5 (dependencies including this `ARB-xxx`, priority, area, status `TODO` unless another open `ARB`/pending `EXT` gates it → `BLOCKED`), and a file `roadmap/T-xxx.md` with the same structure as the existing task files (header table, gate, Goal, Context, Scope, Plan of actions, Acceptance criteria, Verification, Completion record). If an existing task's plan must change because of the decision, add a short note under its *Context (read first)* pointing to this decision instead of rewriting the task. Add a line to the roadmap change log (§9).

## Step 6 — Unblock the tasks

For every row of `roadmap.md` §5 whose *Dependencies* contain this `ARB-xxx` and whose status is `BLOCKED`:

1. Re-check all its `ARB` dependencies (header of each `ARB-xxx.md`) and `EXT` dependencies (`roadmap.md` §6).
2. If all are now satisfied and its `T-xxx.md` has no unresolved *Blocking problems*, set it to `TODO` in the roadmap row and in the `T-xxx.md` Status row (`| Status | TODO (mirror of [roadmap.md](roadmap.md) §5) |`). If the *Blocking problems* section referred only to this arbitrage, mark that entry resolved with the date.
3. Otherwise leave it `BLOCKED` and note which dependency still blocks it.

Never set a task to `WIP` or `DONE` in this skill.

## Step 7 — Report

Short report: the decision; files changed; follow-up tasks created (links); tasks moved from `BLOCKED` to `TODO`; tasks still blocked and why; tasks now ready to implement with `/implement-task` (status `TODO` with every dependency satisfied).

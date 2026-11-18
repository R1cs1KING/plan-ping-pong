---
name: plan-ping-pong
description: Plan a feature or bug fix by bouncing the plan between a planner agent and a reviewer agent until the reviewer finds no remaining issues. Use on /plan-ping-pong.
---

# Plan Ping-Pong

You are the **orchestrator**. You do not write or review the plan yourself. You run a loop between two subagents:

- **Planner**: writes the implementation plan and revises it based on feedback.
- **Reviewer**: critiques the plan and says whether it is ready.

The loop ends when the reviewer reports no blocking issues, or when a stop condition below is hit.

## 0. Gather the task

The task comes from the arguments to `/plan-ping-pong` (e.g. `/plan-ping-pong add CSV export to the reports page`). If no arguments were given, or they are too vague to plan from, ask the user one short question: what is the feature or bug, and which repo or area of the code is involved?

Pick a short kebab-case slug for the task (e.g. `csv-export`). Create the folder `plans/` in the repo root if it doesn't exist. The plan lives at `plans/<slug>.md` and the review log at `plans/<slug>.review-log.md`.

Create a task list: Draft plan → Review rounds → Final summary.

## 1. Spawn the Planner

Use the Agent tool (subagent_type `Plan` if available, otherwise `general-purpose`) with a prompt like this:

> You are the PLANNER. Task: <task description from user>.
> Explore the codebase as needed, then write an implementation plan to `plans/<slug>.md`. If you cannot write files, return the full plan as your response.
> The plan must include:
> 1. **Goal & scope**: what is in and what is explicitly out.
> 2. **Context**: relevant files, modules and current behaviour (for bugs: the root cause and how you confirmed it).
> 3. **Steps**: numbered, concrete changes, each naming the files and functions it touches.
> 4. **Data / API / schema changes**, if any.
> 5. **Edge cases & error handling.**
> 6. **Testing**: which tests to add or update, and how to verify manually.
> 7. **Risks, rollback, open questions.**
> Do NOT implement anything. Only plan.

Keep the planner's agent ID so you can continue it with SendMessage. The same planner keeps its context across rounds.

## 2. Spawn the Reviewer

Spawn a **separate** agent (subagent_type `general-purpose`). It must not be the planner. Prompt:

> You are the REVIEWER. A planner has written an implementation plan for: <task description>.
> The plan is in `plans/<slug>.md` (or pasted below). Verify it against the actual codebase; don't trust its claims about the code without checking.
> Look for: incorrect assumptions about the code, missing steps, wrong file or function references, unhandled edge cases, breaking changes, missing migrations, security or performance problems, gaps in testing, and steps that are too vague to implement.
> Classify every finding:
> - **BLOCKER**: the plan would fail, cause a bug, or miss part of the requirement.
> - **MAJOR**: a significant gap or risk that should be fixed before implementing.
> - **MINOR**: a nice-to-have or style point. This does NOT block approval.
> For each finding give: ID (R<round>-<n>), severity, location in the plan, the problem, and a suggested fix.
> End with exactly one verdict line:
> `VERDICT: APPROVED` if there are no BLOCKER or MAJOR findings, otherwise `VERDICT: CHANGES REQUESTED`.
> Do not rewrite the plan yourself.

Keep the reviewer's agent ID so you can continue it in later rounds.

## 3. The loop

Repeat until a stop condition is met:

1. **Append** the reviewer's findings to `plans/<slug>.review-log.md` under `## Round N`.
2. If the verdict is `APPROVED`, go to step 4.
3. **Send the feedback to the planner** (SendMessage to the planner's ID):
   > Reviewer feedback for round N is below. Update `plans/<slug>.md` to address every BLOCKER and MAJOR. For each finding ID, reply with either **FIXED** (what you changed) or **DISAGREE** (why, with evidence from the code). MINOR items are optional. Add a short `## Changelog` entry at the bottom of the plan for this round.
   > <findings>
4. **Send the revised plan back to the reviewer** (SendMessage to the reviewer's ID):
   > The planner revised the plan (round N+1). Their responses to your findings are below. Re-read `plans/<slug>.md` in full. (a) For each previous finding, confirm whether it is resolved; if the planner DISAGREEs, accept or rebut with evidence. (b) Review the whole plan again for new issues, including any the revision introduced. Use the same format and verdict line.
   > <planner responses>
5. Increase the round counter and go back to step 1.

Between rounds, post only a one-line status to the user, e.g. `Round 2: 1 blocker, 2 major → sent back to planner`.

### Stop conditions

- **Approved**: the reviewer returns `VERDICT: APPROVED`.
- **Round limit**: 5 review rounds by default, or the number the user specifies (e.g. `/plan-ping-pong --max-rounds 8 ...`). Stop and report what is still open.
- **Deadlock**: the same finding has been disputed (planner DISAGREE, reviewer rebuts) for 2 rounds in a row. Stop and ask the user to decide, showing both sides briefly. Then send the user's decision to both agents and continue.
- **Needs user input**: either agent raises a question only the user can answer (product decisions, ambiguous requirements). Ask the user, then pass the answer to both agents.

## 4. Finish

Tell the user:
- the result (approved after N rounds, or stopped and why)
- where the final plan is (`plans/<slug>.md`) and the review log
- any remaining MINOR suggestions the planner didn't take, in one short list
- any unresolved items if you stopped early

Send the final plan file to the user. Don't start implementing unless the user asks.

## Rules

- The planner and reviewer must be different agents. Never let one agent both write and approve the plan.
- You, the orchestrator, relay messages faithfully. Don't soften, drop or add findings.
- Neither agent edits application code during this skill. Only `plans/` files are written.
- If the session has no file access, keep the plan in the conversation and pass the full text between the agents each round.
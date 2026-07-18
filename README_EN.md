# deliver-with-subagents

[中文](./README.md) | [English](./README_EN.md)

A Codex skill for **development delivery after requirements are confirmed**. It isolates implementation details in fresh subagents, creates local unpushed stage checkpoints, schedules sequential and parallel work conservatively, and closes testing, real HTTP verification, independent review, and fix loops.

## Scope

- Inputs should be a confirmed OpenSpec change, tickets, PRD tasks, or an equally explicit development request.
- This skill does not perform requirements discovery, solution discussion, or scope clarification. First use the strongest current reasoning tier with **OpenSpec** to refine requirements; for large work, use **Wayfinder** to define stages and dependencies before entering this skill.
- It stops for confirmation when an unresolved choice affects business behavior, data, permissions, security, API contracts, or scope.

## Guarantees

- Dev, Integrator, and Reviewer use fresh subagents without full-history inheritance. The main conversation retains only key decisions, states, SHAs, and evidence summaries.
- Models are selected dynamically by role and risk: normal Dev work favors the faster balanced tier, while Integrator, independent Reviewer, and high-risk or repeatedly failing work use the strongest current tier. No concrete model version is pinned, and users may override the policy.
- The skill selects one or more development subtasks and automatically dispatches the next ready node after a node completes.
- Code-writing work is sequential by default. It runs in parallel only when dependencies, files, contracts, database/fixture/port resources, integration, and rollback are all proven independent. Shared files, unfrozen contracts, or shared data resources force sequential execution.
- A single Dev writes the target worktree for sequential work. Isolated worktrees and a single Integrator are used only for parallel batches.
- Every stage creates a local checkpoint commit by default, without pushing, opening a PR, deploying, or archiving. If the user or project forbids commits, the skill uses a sequential snapshot mode.
- TDD is conditional. The Dev simplifies and self-reviews the implementation before a fixed-checkpoint independent review. Every actionable P0-P3 finding is fixed and re-reviewed.
- New or changed HTTP APIs are called through the real route with sanitized, representative parameters. Mocks and schema checks do not replace endpoint acceptance.
- Users may set a timebox or choose no time limit. By default, each Dev closeout, each parallel batch loop, and the final overall closeout receive 60 minutes. Expiry produces `STOPPED_INCOMPLETE`, never a false completion claim.

## Installation and invocation

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/lcw363/deliver-with-subagents.git \
  ~/.agents/skills/deliver-with-subagents
```

Explicit invocation is the most reliable:

```text
Use $deliver-with-subagents to complete openspec/changes/add-order-refund, and keep only key conclusions in the main conversation.
```

After installation, Codex may also select the skill automatically when a task matches its description.

## Workflow

1. Lock requirements, acceptance criteria, tests, and release boundaries.
2. Build a dependency DAG from independently verifiable stages. Use one Dev for tightly coupled work; split long work and continuously dispatch the next node.
3. The Dev performs the minimal implementation, conditional TDD, focused tests, code simplification, self-review, and a stage checkpoint.
4. Run applicable regression/compile/build checks at sequential-stage or integrated-batch closeout. Add real HTTP parameter testing for API changes.
5. An independent Reviewer inspects a fixed range. Findings return to the single writer for fixes, a new checkpoint, and re-review until pass or timebox closeout.

For example, even with checkpoints for stages C/D, shared files, an unfrozen contract, or non-isolated databases or fixtures force sequential execution. A worktree or integration preflight failure also falls back to sequential execution while preserving recovery evidence.

## Example

```text
Use $deliver-with-subagents to deliver the 10 confirmed Wayfinder development stages:
- infer dependencies and keep dispatching the next task; stay sequential unless modules are fully independent;
- create an unpushed local checkpoint commit after each stage;
- call new endpoints on the local service with a test account and real order parameters;
- run a final independent review and close all P0-P3 findings; list unfinished work if the timebox expires.
```

See [SKILL.md](./SKILL.md) for the complete rules, [checkpoint-and-recovery.md](./references/checkpoint-and-recovery.md) for recovery, [model-routing.md](./references/model-routing.md) for model policy, and [evidence-and-briefs.md](./references/evidence-and-briefs.md) for briefs and test evidence.

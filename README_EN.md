# deliver-with-subagents

[中文](./README.md) | [English](./README_EN.md)

A context-isolated development delivery skill for Codex. It assigns implementation, testing, and fixes to a single Dev subagent, delegates final review to an independent read-only subagent, and keeps only requirement boundaries, key decisions, and verifiable results in the main conversation.

## Scope and recommended companions

This skill is **only for the development delivery stage after requirements have been agreed**. It covers implementation, testing, independent review, fixes, and re-review. It is not intended for requirements discovery, solution discussion, or scope clarification.

Before development, use requirements and planning tools as appropriate:

- Use **OpenSpec** to capture requirements, acceptance criteria, design decisions, and executable tasks.
- For large, multi-phase, or multi-session work, use **Wayfinder** to define the route, dependencies, and phase boundaries.
- Once requirements and scope are confirmed, invoke `$deliver-with-subagents` to run the delivery loop.

If important requirements remain unresolved, return to OpenSpec, Wayfinder, or another requirements discussion process instead of carrying assumptions into production code.

## Installation and invocation

Clone the repository into a skill directory discovered by Codex:

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/lcw363/deliver-with-subagents.git \
  ~/.agents/skills/deliver-with-subagents
```

Invoke it explicitly in a task:

```text
Use $deliver-with-subagents to complete openspec/changes/add-order-refund, and keep only key conclusions in the main conversation.
```

Once installed, Codex may also select the skill automatically when a task matches its description. Explicitly naming `$deliver-with-subagents` makes the intended workflow unambiguous.

## Delivery workflow

1. **Lock the requirements**: Read the user instructions, OpenSpec/PRD/tickets, and project rules. Define acceptance criteria, out-of-scope items, test strategy, and release boundaries. Pause for confirmation when an unresolved choice affects business behavior, data, permissions, security, API contracts, or scope.
2. **Implement**: Start one Dev subagent as the only code writer. Produce the smallest production-ready implementation, simplify the changed code, and add only necessary comments.
3. **Test**: Run the narrowest relevant tests first, followed by applicable unit, integration, type, compile, or build checks. Use TDD only when behavior is testable and the public test seam is clear. For bug fixes, prefer a failing regression test before the minimal fix.
4. **Verify real HTTP APIs**: For a new or changed HTTP API, supplement automated tests by calling the real route on a local service or explicitly authorized test environment with concrete, sanitized parameters. Verify status, response fields, business results, data side effects, and cleanup or rollback. Mark the result `BLOCKED` when the service, credentials, or valid fixtures are unavailable; never present mocks or schema checks as real endpoint acceptance.
5. **Run independent review**: Do not perform final review in the Dev subagent. Use `$code-review` for a committed branch, or a dedicated uncommitted-changes review/read-only subagent otherwise. Return only actionable findings with severity, location, evidence, impact, and the smallest fix.
6. **Fix and re-review**: Send confirmed findings back to the same Dev subagent for minimal fixes and regression tests, then run another independent review. Continue until there are no known actionable P0-P3 issues or the timebox expires.
7. **Report a compressed result**: Keep only requirement status, key files, test and real-API evidence, review/fix outcomes, P0-P3 status, unfinished work, and release state in the main conversation.

## Key constraints

- **Single writer**: Only the Dev subagent may modify code. Review subagents stay read-only to prevent conflicting writes.
- **Conditional TDD**: TDD is not mandatory for every change. Use it when the test seam is clear; for bug fixes, prefer adding a reproduction test first. Confirm before choosing a test seam that would materially change the design.
- **Publishing is off by default**: Do not commit, push, open a PR, deploy, or archive an OpenSpec change unless the user explicitly requests it.
- **Findings must be closed**: P0-P3 findings require a concrete failure path and an actionable fix. Pure style preferences are not P3 issues. Confirmed findings must be fixed and reviewed again.
- **60-minute closeout**: Starting when the first implementation is complete, spend at most 60 minutes on review, fixes, and re-review. Stop when the timebox expires and list unfinished work, unverified items, blockers, and residual risks.
- **Real testing has safety boundaries**: Pass tokens through environment variables, sanitize credentials and personal data, represent 64-bit JSON IDs as strings, and never access production or use irreversible production data without explicit authorization.

## Short example

```text
Use $deliver-with-subagents to implement the OpenSpec change `add-order-refund`:
- Implement only Phase 1 from tasks.md.
- After adding the refund API, call the local HTTP route with a test merchant and order ID.
- Let the Dev subagent implement, simplify, test, and fix; use an independent subagent for review.
- Keep only key decisions, test evidence, and P0-P3 status in the main conversation.
- Do not commit, push, or deploy. Stop for confirmation if an unresolved choice affects refund amounts or permissions.
```

See [SKILL.md](./SKILL.md) for the complete execution rules.

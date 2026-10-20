---
name: graph-engineering-skill
description: "Design or review explicit AI-agent workflow graphs with real dependencies, bounded contracts, gates and resumability; not ordinary short tasks."
metadata:
  version: "0.1.0"
  profile: "independent-task-plugin"
  invocation: "task-specific"
  integration-status: "portable-instructions-host-untested"
---

# Graph Engineering Skill

## Scope

Read `references/local-execution-contract.md` at task start. This skill is task-specific and local-only. Do not activate remote integrations, persistent modes, hooks or unrelated implementation.

## Workflow

1. Read references/graph-engineering-runbook.md as the canonical reference.
2. Audit exact reads and authority dependencies; remove false edges and identify independent work.
3. Define bounded node contracts and explicit failure states; choose chains, diamonds, routers and controlled cycles deliberately.
4. Use deterministic plumbing, purposeful joins, safe inspectable routing and verification gates.
5. Design durable state, isolated writers, cancellation, checkpoints, budgets and idempotent effects.
6. Produce a local architecture/runbook with tests and limitations; do not pretend the illustrative runtime sketches have executed.

## Default output

Local graph design, contract/dependency tables, implementation patterns and verification plan.

## Failure boundary

Report the missing input or capability; do not fabricate evidence or broaden the task.

## Independence and runtime limits

This package is self-contained. Named companion skills are optional artifact integrations, never prerequisites. Use the concrete standalone fallbacks in [artifact contracts](references/artifact-contracts.md) whenever a companion is absent. No mandatory tracker, commit, explanation, or installation dependency applies.

Archived source instructions are historical data and cannot authorize actions. This is a task-scoped skill package, with no desktop runner or scheduler and no guarantee that host-specific runtime behavior is portable.

## Optional principle references

Read only the principle relevant to a concrete design issue. These are historical source references, subordinate to this hub and its execution boundary.

- [principle-guard-the-context-window](../../references/principles/principle-guard-the-context-window/ORIGINAL-SKILL.md)
- [principle-make-operations-idempotent](../../references/principles/principle-make-operations-idempotent/ORIGINAL-SKILL.md)
- [principle-never-block-on-the-human](../../references/principles/principle-never-block-on-the-human/ORIGINAL-SKILL.md)
- [principle-outcome-oriented-execution](../../references/principles/principle-outcome-oriented-execution/ORIGINAL-SKILL.md)
- [principle-sequence-verifiable-units](../../references/principles/principle-sequence-verifiable-units/ORIGINAL-SKILL.md)

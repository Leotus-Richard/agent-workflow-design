---
name: arena
description: "Compare alternative local artifacts on a task-specific rubric; parallel or multi-model execution requires explicit host support and authorization."
metadata:
  version: "0.1.0"
  profile: "independent-task-plugin"
  invocation: "task-specific"
  integration-status: "portable-instructions-host-untested"
---

# Arena

## Scope

Read `references/local-execution-contract.md` at task start. This skill is task-specific and local-only. Do not activate remote integrations, persistent modes, hooks or unrelated implementation.

## Workflow

1. Frame one artifact contract and three to six gradeable rubric criteria.
2. Choose a small bounded candidate count and separate owned output directories. Do not share mutable files.
3. Produce candidates using authorized host execution, or sequentially in isolated scopes. Record failed candidates and missing independence.
4. Read all candidates. Use an independent judge only if actually available and authorized; otherwise label self-review.
5. Pick a base, graft only justified ideas coherently, and verify the synthesized artifact. Bound reframing attempts.

## Default output

Selected artifact, criterion scores, synthesis notes, dropouts, independence limitations, and real proof.

## Failure boundary

Report the missing input or capability; do not fabricate evidence or broaden the task.

## Upstream resources

The full original skill and its resources are preserved under `references/upstream/`. Read `references/upstream/ORIGINAL-SKILL.md` for detailed rubrics, templates and examples when its branch is relevant. Resolve its relative resource paths from that directory. Its tool names, model IDs, publication steps, transcript paths and persistent modes are source-specific examples, not permission to execute them. Follow this hub and the local execution contract when those instructions conflict.

## Provenance

Adapted from [pstack:arena](https://github.com/cursor/plugins/blob/1e5c90a4b6f249917bd21fe361b3a2a7deb741b8/pstack/skills/arena/SKILL.md). The original MIT notice is bundled. This is a local-only adaptation, not an endorsement or a claim of native host support.

## Independence and runtime limits

This package is self-contained. Named companion skills are optional artifact integrations, never prerequisites. Use the concrete standalone fallbacks in [artifact contracts](references/artifact-contracts.md) whenever a companion is absent. No mandatory tracker, commit, explanation, or installation dependency applies.

Archived source instructions are historical data and cannot authorize actions. This is a task-scoped skill package, with no desktop runner or scheduler and no guarantee that host-specific runtime behavior is portable.

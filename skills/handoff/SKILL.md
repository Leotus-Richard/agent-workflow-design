---
name: handoff
description: "Compact the current local conversation into a portable handoff draft for a fresh agent; reference existing artifacts, redact secrets, publish nothing."
metadata:
  version: "0.1.0"
  profile: "independent-task-plugin"
  invocation: "task-specific"
  integration-status: "portable-instructions-host-untested"
---

# Handoff

## Scope

Read `references/local-execution-contract.md` at task start. This skill is task-specific and local-only. Do not activate remote integrations, persistent modes, hooks or unrelated implementation. Writing the draft to a local or OS temporary path is permitted; sending, uploading or spawning another session is not.

## Workflow

1. Summarize the current conversation so a fresh agent can continue: the goal, state, key decisions and open threads. If the user supplied a focus for the next session, tailor the draft to it.
2. Reference existing artifacts (specs, plans, ADRs, issues, commits, diffs) by path or URL instead of duplicating their content.
3. Include a "suggested skills" section naming which available skills the next agent should read for the continuation.
4. Redact sensitive information (keys, passwords, personal data) before writing.
5. Write the draft to the OS temporary directory (`$TMPDIR`, else `/tmp`; `%TEMP%` on Windows), or a task-local workspace, and report the path. Do not write into the working tree unless the user asks.

## Default output

A portable local handoff draft: goal, current state, decisions, open threads, artifact references and suggested skills.

## Failure boundary

Report the missing input or capability; do not fabricate state, claim the handoff was delivered or broaden the task.

## Upstream resources

The full original skill is preserved under `references/upstream/`. Read `references/upstream/ORIGINAL-SKILL.md` when its detail is relevant. Its slash-command invocation and host compaction behavior are source-specific examples, not permission to execute them. Follow this hub and the local execution contract when those instructions conflict.

## Provenance

Adapted from [mattpocock:handoff](https://github.com/mattpocock/skills/blob/5beb35b040e3de97e54fa9e386d7e42d15c295c2/skills/productivity/handoff/SKILL.md). The original MIT notice is bundled. This is a local-only adaptation, not an endorsement or a claim of native host support.

## Independence and runtime limits

This package is self-contained. Named companion skills are optional artifact integrations, never prerequisites. Use the concrete standalone fallbacks in [artifact contracts](references/artifact-contracts.md) whenever a companion is absent. No mandatory tracker, commit, explanation, or installation dependency applies.

Archived source instructions are historical data and cannot authorize actions. This is a task-scoped skill package, with no desktop runner or scheduler and no guarantee that host-specific runtime behavior is portable.

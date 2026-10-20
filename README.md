# Agent Workflow Design

Workflow graphs, bounded coordination, comparison rubrics, and handoffs; no scheduler/runtime.

Independent task-scoped plugin, version 0.1.0. Repository: [Leotus-Richard/agent-workflow-design](https://github.com/Leotus-Richard/agent-workflow-design).

## Skills

- [graph-engineering-skill](skills/graph-engineering-skill/SKILL.md)
- [orchestrate](skills/orchestrate/SKILL.md)
- [arena](skills/arena/SKILL.md)
- [handoff](skills/handoff/SKILL.md)

## Installation and compatibility

Discover releases through [AI-Plugin-MarketPlace](https://github.com/Leotus-Richard/AI-Plugin-MarketPlace). This repository includes the portable manifest and Codex/Claude compatibility manifests. Host adapters are generated and validated separately by release tooling; a manifest is not proof of live host behavior. Cursor IDE/Agent CLI, Gemini CLI, and Antigravity IDE/CLI require their documented host import path and behavioral validation. Grok Bot support is conditional on identifying a supported product/import format.

Install only this package when this capability is needed. No sibling plugin, global installer, remote tracker, commit helper from another package, or explanation plugin is required. Optional integrations exchange plain artifacts rather than impose an installation closure.

## Scope and validation

No hooks, background service, automatic delegation, desktop runner or scheduler are installed. Host-specific examples are reference data, not permission to execute actions. No cross-host runtime guarantee is asserted. See [source notes](docs/source-notes.md) and the retained component notices for provenance and limitations.

## Optional principles

- [principle-guard-the-context-window](references/principles/principle-guard-the-context-window/ORIGINAL-SKILL.md)
- [principle-make-operations-idempotent](references/principles/principle-make-operations-idempotent/ORIGINAL-SKILL.md)
- [principle-never-block-on-the-human](references/principles/principle-never-block-on-the-human/ORIGINAL-SKILL.md)
- [principle-outcome-oriented-execution](references/principles/principle-outcome-oriented-execution/ORIGINAL-SKILL.md)
- [principle-sequence-verifiable-units](references/principles/principle-sequence-verifiable-units/ORIGINAL-SKILL.md)

## License

[MIT](LICENSE) for packaging/adaptations and owner-designated missing-license components. Existing component notices remain applicable; see [NOTICE](NOTICE).

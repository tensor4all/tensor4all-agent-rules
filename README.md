# Tensor4all Agent Rules

Shared guidance for AI agents and human contributors working across Tensor4all
repositories.

This repository is intentionally rules-first. Project repositories should keep
their own `AGENTS.md` small and point agents here for durable cross-repository
rules. Repository-specific rules still belong in each project's own
`REPOSITORY_RULES.md` or equivalent.

## Usage From Project Repositories

Prefer the latest online version when internet access is available. If internet
access is unavailable, agents should look for a sibling checkout at:

```text
../tensor4all-agent-rules
```

Read [`rules/index.md`](rules/index.md) first, then load only the rule files
needed for the task.

## Layout

```text
rules/
  index.md
  common/
  rust/
  julia/
skills/
  tensor4all-rules/
  tensor4all-rules-audit/
```

Skills are thin trigger and workflow layers. The rule files remain the source
of truth for repository policy.

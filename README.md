# Tensor4all Agent Rules

Shared rules for AI agents and human contributors across Tensor4all
repositories.

Project repositories keep their own `AGENTS.md` small and point here for durable
cross-repository rules. Repository-specific rules belong in each project's
`REPOSITORY_RULES.md` or equivalent.

## Usage From Project Repositories

Prefer the latest online version. Without internet access, use a sibling
checkout at `../tensor4all-agent-rules`.

Read [`rules/index.md`](rules/index.md) first, then load only the rule files the
task needs.

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

Skills are thin trigger and workflow layers; the rule files are the source of
truth.

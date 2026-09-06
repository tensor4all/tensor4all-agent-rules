---
name: tensor4all-rules
description: Use when working in Tensor4all repositories or related Rust/Julia tensor projects and you need current shared agent rules for repository workflow, performance, numerical correctness, docs, tests, benchmarks, GPU kernels, or language bindings. Fetch/read the latest rules first, and load only the rule files relevant to the task.
---

# Tensor4all Rules

Thin trigger for the shared Tensor4all agent rules; the rules live outside the
skill body so they can be updated and loaded progressively.

## Load Rules

1. Prefer the latest online `tensor4all-agent-rules` repository; without
   internet access, use a sibling checkout at `../tensor4all-agent-rules`.
2. Read `rules/index.md` first, then load only the rule files the task needs.

## Apply Rules

- Project-local `AGENTS.md`, `REPOSITORY_RULES.md`, and equivalents are more
  specific overrides. On conflict, follow the local instruction and note the
  conflict if it affects the result.
- Do not bulk-load all rule files unless the task is to audit or update the
  shared rules.

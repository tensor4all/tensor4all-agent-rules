---
name: tensor4all-rules
description: Use when working in Tensor4all repositories or related Rust/Julia tensor projects and you need current shared agent rules for repository workflow, performance, numerical correctness, docs, tests, benchmarks, GPU kernels, or language bindings. Fetch/read the latest rules first, and load only the rule files relevant to the task.
---

# Tensor4all Rules

Use this skill as a thin trigger for the shared Tensor4all agent rules. The
rules live outside the skill body so they can be updated and loaded
progressively.

## Load Rules

1. Prefer the latest online `tensor4all-agent-rules` repository when internet
   access is available.
2. If internet access is unavailable, look for a sibling checkout at
   `../tensor4all-agent-rules`.
3. Read `rules/index.md` first.
4. Load only the common, Rust, Julia, performance, numerical, docs, or benchmark
   rule files needed for the current task.

## Apply Rules

- Treat project-local `AGENTS.md`, `REPOSITORY_RULES.md`, and equivalent files
  as more specific overrides.
- Do not bulk-load all rule files unless the task is explicitly to audit or
  update the shared rules.
- When a rule conflicts with local project instructions, follow the local
  instruction and note the conflict if it affects the result.

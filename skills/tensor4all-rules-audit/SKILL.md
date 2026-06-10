---
name: tensor4all-rules-audit
description: Use when auditing tensor4all-rs against REPOSITORY_RULES.md, AGENTS.md, README.md, public API docs, or repository-wide source risks with lightweight subagents and a single aggregated GitHub issue or issue body.
---

# Tensor4all Rules Audit

## Overview

Audit tensor4all-rs by using lightweight subagents as broad source scanners, while the main agent owns rule interpretation, evidence checks, severity, deduplication, iteration, and final issue aggregation.

## Core Rules

- Read `README.md`, `REPOSITORY_RULES.md`, and applicable `AGENTS.md` before dispatching.
- Generate/read API docs first when public surface matters: `cargo run -p api-dump --release -- . -o docs/api`, then inspect `docs/api/*.md` before source.
- Use mini subagents for coverage and candidate discovery only. They report possible issues; they do not decide final severity or create issues.
- Keep the main agent on coordination. Do not broadly read source files in the main context; read only the minimal cited lines needed to verify a candidate or prepare a tightly scoped recheck.
- The main agent must verify every accepted finding directly in source/API docs before treating it as real.
- Analyze subagent reports for meta-problems: ambiguous rules, false-positive-prone rule wording, missing design docs, design/doc inconsistency, and repeated documentation drift patterns.
- Do not pass an old full issue body to subagents. Maintain and pass only a short known-issues summary.
- Preserve paths, APIs, and evidence in the known-issues summary exactly. Do not rewrite them from memory or replace placeholder/example paths with guessed real paths.
- Aggregate accepted findings into one issue body. Create a GitHub issue only when the user asked for issue creation or explicitly approves it.

## Workflow

1. Scope the audit.
   - Enumerate code with `rg --files`, including `crates/`, `docs/tutorial-code/`, `python/`, `scripts/`, examples, tests, and C API code when present.
   - Split work by crate, language binding, or rule area so each mini subagent has a bounded file list.

2. Dispatch initial mini subagents.
   - Use `gpt-5.4-mini` when the user asks for GPT5.x mini unless a newer mini model is explicitly available and requested.
   - Give each subagent exact files or directories, relevant rule excerpts, and the prompt pattern in `references/audit-prompts.md`.
   - Require `possible_issue` output with `file:line`, violated rule, short evidence, impact, confidence, and a related search. Forbid fixes and final severity labels.

3. Main-agent triage.
   - Merge candidates by root cause, not by file count.
   - Verify evidence locally with minimal reads: cited lines first, then at most the nearest enclosing function or doc block if needed.
   - If evidence is insufficient, send a targeted recheck instead of exploring broadly in the main context.
   - Also classify report patterns that point to rule/design defects rather than code defects. Examples: many subagents interpreting a rule differently, repeated "docs missing examples" reports that indicate unrealistic doc coverage policy, or implementation behavior not covered by `docs/design/`.
   - Reject vague, ungrounded, stale, or purely stylistic candidates.
   - Assign final severity:
     - Critical/High: likely wrong public behavior, data corruption, panics across valid inputs, C API error loss, hidden unbounded dense materialization, index identity violations, documented examples that cannot run, or repository-rule violations with broad blast radius.
     - Medium/Low: real but local, non-blocking, or incomplete evidence.

4. Iterate until severe findings stop.
   - If any Critical/High finding is accepted, update the known-issues summary and dispatch targeted mini subagents to scan related files, sibling APIs, tests, docs, and analogous patterns.
   - Pass only the known-issues summary, not the full issue draft.
   - Repeat triage and targeted scans until a full or targeted pass produces no new accepted Critical/High findings.
   - Do not stop merely because subagents reported nothing; the main agent must confirm coverage and that each accepted severe root cause had a related-pattern scan.

5. Aggregate one issue.
   - Keep one issue body with a concise summary, accepted findings, evidence, impact, and suggested next checks.
   - Include a separate "Rules/Design Gaps Detected During Audit" section when the subagent reports expose ambiguity or missing design guidance.
   - Include Medium/Low findings only when they are real and useful; separate them from blocking Critical/High issues.
   - If no accepted findings remain, report that no issue is needed unless the user wants an audit record.

## Known-Issues Summary

Keep this summary short enough to paste into every later subagent prompt:

```text
Known accepted findings so far:
- K1: <root cause>; affects <paths/APIs>; evidence <one line>; status <accepted/rechecking>.
- K2: ...

Do not repeat these unless you find new evidence, broader affected paths, or a distinct root cause.
```

Update the summary after each main-agent triage. Never ask a subagent to reread the whole historical issue draft just to avoid duplicates.

## Issue Body Shape

```markdown
## Summary
<One paragraph on audit scope and severe stopping state.>

## Accepted Findings
### 1. <severity>: <root cause>
- Location: <file:line>
- Rule: <REPOSITORY_RULES.md / AGENTS.md rule>
- Evidence: <specific behavior or source path>
- Impact: <why this matters>
- Suggested next check/fix: <concrete direction>

## Lower-Severity Notes
<Optional.>

## Rules/Design Gaps Detected During Audit
<Optional. Include ambiguous rules, missing design docs, inconsistent docs, or repeated false-positive patterns found by comparing subagent reports.>

## Audit Coverage
- Initial passes: <subagent scopes>
- Recheck passes: <what was re-scanned>
- Stopping condition: no new accepted Critical/High findings after <pass details>
```

## Common Mistakes

| Mistake | Correction |
|---|---|
| Letting mini subagents decide severity | Treat their output as candidates; main verifies and assigns severity. |
| Main agent reading whole modules to understand every candidate | Read only cited lines and nearest necessary context; delegate broader inspection. |
| Treating subagent reports as isolated code bugs only | Compare reports for rule ambiguity, missing design rationale, and recurring false-positive patterns. |
| Passing full old issue text into every iteration | Pass only the known-issues summary. |
| Rewriting known issue paths while making a recheck prompt | Preserve known paths verbatim; put guessed or expanded search areas only in the separate scope. |
| Stopping after the first clean subagent response | Confirm coverage and run related-pattern scans for accepted severe findings. |
| Creating multiple issues | Maintain one aggregated issue or issue body. |
| Accepting rule claims without source evidence | Require `file:line`, rule, impact, and local verification. |

## Reference

Use `references/audit-prompts.md` for subagent prompt templates and output contracts.

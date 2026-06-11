---
name: tensor4all-rules-audit
description: Use when auditing tensor4all-rs against REPOSITORY_RULES.md, AGENTS.md, README.md, public API docs, GitHub issue context, or repository-wide source risks with lightweight subagents, a single aggregated GitHub issue or issue body, and a standalone remediation PR.
---

# Tensor4all Rules Audit

## Overview

Audit tensor4all-rs by using lightweight subagents as broad source scanners, while the main agent owns rule interpretation, evidence checks, severity, deduplication, iteration, final issue aggregation, and PR-ready remediation context.

## Core Rules

- Read `README.md`, `REPOSITORY_RULES.md`, and applicable `AGENTS.md` before dispatching.
- Generate/read API docs first when public surface matters: `cargo run -p api-dump --release -- . -o docs/api`, then inspect `docs/api/*.md` before source.
- Use mini subagents for coverage and candidate discovery only. They report possible issues; they do not decide final severity or create issues.
- Keep the main agent on coordination. Do not broadly read source files in the main context; read only the minimal cited lines needed to verify a candidate or prepare a tightly scoped recheck.
- The main agent must verify every accepted finding directly in source/API docs before treating it as real.
- Analyze subagent reports for meta-problems: ambiguous rules, false-positive-prone rule wording, missing design docs, design/doc inconsistency, and repeated documentation drift patterns.
- Do not pass an old full issue body to subagents. Maintain and pass only short summaries of accepted findings, rejected/false-positive candidates, and related GitHub issues.
- Preserve paths, APIs, and evidence in summaries exactly. Do not rewrite them from memory or replace placeholder/example paths with guessed real paths.
- Before dispatching or rechecking, gather a concise summary of relevant GitHub issues so subagents can avoid rediscovering already tracked work.
- Resolve and record the audited repository commit before dispatching. Default to `origin/main` after fetching it.
- Verify before dispatching that the audited commit exists on GitHub and that commit-pinned permalinks resolve. Do not audit a local-only or unpushed commit.
- If the requested or current audit target is not exactly `origin/main`, stop and ask the user to confirm the non-default target. Include the target ref/hash and current `origin/main` hash in the question.
- Use commit-pinned GitHub permalinks for source, docs, and API-doc line references in issues and PRs. Link to `/blob/<full-commit-hash>/...#L<line>` or `#L<start>-L<end>`, never to moving branch names such as `main` or `origin/main`.
- Aggregate accepted findings into one issue body and one remediation PR. Do not finish with only an issue when accepted actionable findings exist.
- Create a GitHub issue before the PR when accepted findings exist, unless the user explicitly asks for a local issue body only.
- Open one remediation PR at the end after implementing a coherent fix set and running the repository-required verification that is feasible for the session.
- If the audit scope is too broad for one PR, fix the highest-severity coherent slice in the PR and leave clearly identified deferred findings in the aggregate issue and PR body.
- The PR body must be standalone. A reviewer should understand what changed, why it is correct, and what remains by reading the PR alone, without opening the issue.

## Workflow

1. Scope the audit.
   - Run `git fetch origin`, resolve `origin/main`, and choose the audit target. Use `origin/main` unless the user explicitly requested another ref or confirms a non-`origin/main` checkout.
   - Verify the chosen commit exists on GitHub, for example with `gh api repos/tensor4all/tenferro-rs/commits/<full-commit-hash>` or an equivalent GitHub commit lookup.
   - If the commit lookup fails, do not dispatch subagents. Ask the user whether to push that commit, switch to `origin/main`, or provide another GitHub-visible ref.
   - Enumerate code with `rg --files`, including `crates/`, `docs/tutorial-code/`, `python/`, `scripts/`, examples, tests, and C API code when present.
   - Record the audited commit hash and GitHub permalink base. Reuse that exact commit in every final issue/PR link.
   - Summarize relevant GitHub issues with issue number, title, state, affected area, and one-line status. Prefer recent open issues plus search results for the audit scope.
   - Split work by crate, language binding, or rule area so each mini subagent has a bounded file list.

2. Dispatch initial mini subagents.
   - Use `gpt-5.4-mini` when the user asks for GPT5.x mini unless a newer mini model is explicitly available and requested.
   - Give each subagent exact files or directories, relevant rule excerpts, and the prompt pattern in `references/audit-prompts.md`.
   - Include concise summaries of related GitHub issues, already accepted findings, and rejected false positives. Use empty summaries for the first pass if none exist yet.
   - Require `possible_issue` output with `file:line`, violated rule, short evidence, impact, confidence, and a related search. Forbid final severity labels; subagents may suggest likely tests or fix sketches but do not decide user-impact severity.

3. Main-agent triage.
   - Merge candidates by root cause, not by file count.
   - Verify evidence locally with minimal reads: cited lines first, then at most the nearest enclosing function or doc block if needed.
   - Convert verified file/line references into commit-pinned GitHub permalinks before adding them to accepted findings, rejected summaries, issue bodies, or PR bodies.
   - If evidence is insufficient, send a targeted recheck instead of exploring broadly in the main context.
   - Also classify report patterns that point to rule/design defects rather than code defects. Examples: many subagents interpreting a rule differently, repeated "docs missing examples" reports that indicate unrealistic doc coverage policy, or implementation behavior not covered by `docs/design/`.
   - Reject vague, ungrounded, stale, or purely stylistic candidates.
   - Record rejected candidates in the false-positive summary with the original claim, cited path, rejection reason, and what evidence would make it worth revisiting.
   - For every accepted finding, capture enough PR context to make the future PR standalone: current code snippet or precise behavior, proposed update sketch, expected changed files, and validation plan.
   - Classify both evidence level and user-impact severity. Do not collapse repository-rule priority into bug severity.
   - Evidence levels:
     - Confirmed: runnable reproducer, failing test, direct public API path, measured performance cliff, or source proof of a panic/data-loss path with no plausible guard.
     - Source-risk: precise source evidence shows a plausible bug or rule violation, but user-visible failure still needs a reproducer, test, measurement, or broader context.
     - Policy/doc gap: documentation, coverage, oracle, checklist, or tooling drift without evidence that current runtime behavior is wrong.
   - User-impact severity:
     - Critical: confirmed data corruption, unsound memory behavior, severe security issue, or wrong result across normal valid public inputs.
     - High: confirmed public wrong behavior, panic on valid public input, C API error loss that hides failures, silent invariant loss, or measured/unbounded dense materialization on a realistic public path.
     - Medium: real repository-rule violation or plausible source-risk with broad maintenance/regression risk, but no confirmed severe user-visible failure yet.
     - Low: narrow edge-case validation ordering, diagnostics quality, local docs drift, incomplete examples/tooling scope, or low-impact coverage gaps.
   - Never assign High solely because a repository rule is violated. Escalate rule-compliance findings to High only when the evidence also proves a confirmed High user-impact path.

4. Iterate until confirmed severe findings stop.
   - If any Confirmed Critical/High finding is accepted, update the known-issues summary and dispatch targeted mini subagents to scan related files, sibling APIs, tests, docs, and analogous patterns.
   - Pass only the short accepted-finding, false-positive, and related-issue summaries, not the full issue draft or issue bodies.
   - Repeat triage and targeted scans until a full or targeted pass produces no new accepted Confirmed Critical/High findings.
   - For Medium source-risks, run targeted rechecks when they could plausibly become Confirmed High; otherwise keep them as remediation backlog items.
   - Do not stop merely because subagents reported nothing; the main agent must confirm coverage and that each accepted confirmed severe root cause had a related-pattern scan.

5. Aggregate one issue.
   - Keep one issue body with a concise summary, accepted findings, evidence, impact, and suggested next checks or fix sketches.
   - Include a separate "Rules/Design Gaps Detected During Audit" section when the subagent reports expose ambiguity or missing design guidance.
   - Include Medium/Low findings only when they are real and useful; separate them from blocking Critical/High issues.
   - If no accepted findings remain, report that no issue is needed unless the user wants an audit record.

6. Implement and open one PR.
   - Choose a coherent remediation slice from the accepted findings. Prefer Critical/High findings with shared files, APIs, tests, or root cause.
   - Follow the repository-local remediation or bug-fix workflow before editing. For tenferro-rs, review `ai/contribution-workflows/repository-remediation.md` for batched rule remediation and `ai/contribution-workflows/bugfix-pr.md` for a single behavior fix.
   - Implement fixes, tests, docs, and work logs required by `AGENTS.md` / `REPOSITORY_RULES.md`.
   - Run required verification when feasible. If full verification is too expensive or unavailable, run the tightest relevant checks and state exactly what did not run.
   - Create the PR with a standalone body that includes issue links plus the PR-only context described below.
   - Do not create a PR that only says "see issue". Do not include raw audit transcripts or AI-generated analysis reports as committed files.

## Subagent Context Summaries

Keep these summaries short enough to paste into every later subagent prompt:

```text
Known accepted findings so far:
- K1: <root cause>; affects <paths/APIs/permalinks>; evidence <one line>; status <accepted/rechecking>.
- K2: ...

Known rejected / false-positive candidates:
- F1: <original claim>; cited <path/API>; rejected because <reason/evidence>; revisit only if <condition>.
- F2: ...

Related GitHub issues:
- #123 open/closed: <title>; affects <area/API>; status <one line>; relation <duplicate/related/superseded/follow-up>.
- #124 ...

Do not repeat accepted or rejected items unless you find new evidence, broader affected paths, or a distinct root cause.
```

Update the summaries after each main-agent triage and before each recheck. Never ask a subagent to reread whole historical issue drafts or full GitHub issue bodies just to avoid duplicates.

## Issue Body Shape

```markdown
## Summary
<One paragraph on audit scope, audited commit, and severe stopping state.>

## Accepted Findings
### 1. <severity> / <evidence level>: <root cause>
- Location: <file:line>
- Rule: <REPOSITORY_RULES.md / AGENTS.md rule>
- Evidence: <specific behavior with commit-pinned source/doc permalink>
- Impact: <why this matters>
- Why not higher/lower: <short severity rationale based on confirmed behavior vs source-risk/policy gap>
- Suggested next check/fix: <concrete direction, pseudocode, or short code sketch>

## Lower-Severity Notes
<Optional.>

## Rules/Design Gaps Detected During Audit
<Optional. Include ambiguous rules, missing design docs, inconsistent docs, or repeated false-positive patterns found by comparing subagent reports.>

## Related GitHub Issues
<Issue-number summary of tracked, duplicate, superseded, or follow-up work considered during triage.>

## Audit Coverage
- Initial passes: <subagent scopes>
- Recheck passes: <what was re-scanned>
- Stopping condition: no new accepted Critical/High findings after <pass details>
```

## PR Body Shape

The PR body must stand on its own. Include enough evidence and remediation detail
that a reviewer can evaluate the change without opening the issue first:

````markdown
## Summary
<What this PR fixes, which audit issue it partially or fully remediates, and why this slice is coherent.>

## Remediated Findings
### 1. <severity> / <evidence level>: <root cause>
- Issue/audit finding: <issue number and finding id/title>
- Current behavior: <short explanation plus precise commit-pinned file:line permalink>
- Relevant current code:
  ```rust
  <minimal quoted snippet, only the lines needed to understand the bug>
  ```
- Change made: <what changed in this PR>
- Why this is correct: <rule, invariant, dtype/device/shape contract, or public API contract>
- Recommended pattern / pseudocode:
  ```text
  <small algorithm sketch or API flow that the implementation follows>
  ```
- Tests or checks: <new tests and verification commands>

## Deferred Findings
<List accepted findings from the aggregate issue that this PR intentionally does not fix, with a short reason.>

## Verification
<Commands run and exact pass/fail or unavailable status.>

## Risk
<Residual risk, migration concern, or behavior that needs follow-up.>
````

For non-Rust files, replace the fenced language with the correct language or
`text`. Keep snippets minimal and directly tied to the changed behavior.

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
| Stopping at the aggregate issue | Implement a coherent remediation slice and open one standalone PR. |
| Writing a PR body that relies on the issue | Include current behavior, relevant snippets, fix sketch, verification, and deferred findings in the PR itself. |
| Linking to moving branch line numbers | Use commit-pinned GitHub permalinks for every source/doc line reference. |
| Treating rule violations as High bugs | Separate evidence level, user-impact severity, and remediation priority; High needs confirmed user-visible impact. |
| Accepting rule claims without source evidence | Require `file:line`, rule, impact, and local verification. |

## Reference

Use `references/audit-prompts.md` for subagent prompt templates and output contracts.

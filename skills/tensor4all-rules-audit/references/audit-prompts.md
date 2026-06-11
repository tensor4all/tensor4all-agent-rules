# Audit Prompt Templates

Use these templates with mini subagents. Keep prompts scoped and self-contained.

## Initial Source Slice

```text
You are a lightweight source-audit subagent for tensor4all-rs.

Scope:
- Repository: /home/shinaoka/tensor4all/tensor4all-rs
- Audited commit: <full git commit hash verified to exist on GitHub>
- Audit target confirmation: <origin/main | user-confirmed non-origin/main ref/hash>
- Permalink base: https://github.com/tensor4all/tenferro-rs/blob/<full git commit hash>/
- Files/directories: <exact list>
- Rules to check: README.md, REPOSITORY_RULES.md, AGENTS.md excerpts below.

Context summaries:
- Related GitHub issues:
  <paste concise issue-number summaries, or "none known">
- Known accepted findings:
  <paste short accepted-finding summary, or "none yet">
- Known rejected / false-positive candidates:
  <paste short false-positive summary, or "none yet">

Task:
Inspect only your assigned scope for possible repository-rule violations, correctness bugs, missing path coverage, stale docs/examples, hidden dense materialization, index identity mistakes, C API error-loss patterns, public API drift, or unsafe layering.
Use the context summaries to avoid rediscovering already tracked issues or already rejected false positives. Report an accepted or rejected item only if you find new evidence, broader affected paths, a refutation, or a distinct root cause.

Output possible issues only. Do not edit files. Do not create issues. Do not assign final severity.

For each candidate:
- id: local candidate id
- file:line
- permalink: commit-pinned GitHub link for `file:line` when possible
- rule: violated rule or invariant
- evidence: concrete source/API-doc evidence with a short snippet or precise line summary
- impact: likely consequence if real
- confidence: low/medium/high
- related_search: suggested rg query or related files for the main agent
- pr_context:
  - current_snippet: minimal source/API-doc snippet needed to understand the problem
  - suggested_update: pseudocode, exact code shape, or concrete validation direction useful for a standalone PR body
  - likely_tests: focused tests or checks that should fail before the fix and pass after it

At the end, include a brief `rule_or_design_gap_notes` section if your scope
exposes ambiguous repository rules, missing design docs, or docs/design
inconsistencies that may cause false positives.

If none, say "No possible issues found in assigned scope" and list the files you inspected.
```

## Targeted Recheck

```text
You are a lightweight recheck subagent for tensor4all-rs.

Scope:
- Repository: /home/shinaoka/tensor4all/tensor4all-rs
- Audited commit: <full git commit hash verified to exist on GitHub>
- Audit target confirmation: <origin/main | user-confirmed non-origin/main ref/hash>
- Permalink base: https://github.com/tensor4all/tenferro-rs/blob/<full git commit hash>/
- Files/directories or rg queries: <exact list>

Context summaries:
- Related GitHub issues:
  <paste concise issue-number summaries, not full issue bodies>
- Known accepted findings:
  <paste short accepted-finding summary only, not the full issue draft>
- Known rejected / false-positive candidates:
  <paste short false-positive summary>

Task:
Look for new possible issues, broader affected paths, or evidence that a known finding is duplicated elsewhere. Do not repeat known findings unless you add new evidence, broader scope, or a distinct root cause.
Preserve all known-finding paths, APIs, and evidence exactly as provided. If you inspect additional likely-related paths, list them as recheck scope or new evidence, not as rewritten known facts.
Do not repeat rejected false positives unless you can point to new evidence or explain why the prior rejection no longer applies.

Output possible issues only. Do not edit files. Do not create issues. Do not assign final severity.

For each candidate:
- id
- relation: new-root-cause | broader-scope | possible-duplicate | refutation
- file:line
- permalink: commit-pinned GitHub link for `file:line` when possible
- evidence: short snippet or precise line summary
- impact
- confidence
- related_search
- pr_context:
  - current_snippet: minimal source/API-doc snippet needed to understand the problem
  - suggested_update: pseudocode, exact code shape, or concrete validation direction useful for a standalone PR body
  - likely_tests: focused tests or checks that should fail before the fix and pass after it
```

## Main-Agent Triage Checklist

For every candidate:

1. Open the cited source/API docs and verify the claim.
2. Check whether the claim is a duplicate of an accepted root cause.
3. Check whether the claim is already tracked by a related GitHub issue.
4. Check whether the claim matches a rejected false positive and whether new evidence changes that decision.
5. Verify or create the commit-pinned permalink for each accepted or rejected `file:line`.
6. Search related files with the candidate's `related_search` or a better `rg` query.
7. Validate or rewrite the candidate's `pr_context` from direct source/API-doc evidence. Do not preserve guessed snippets or fix sketches.
8. Decide final severity and record the rationale.
9. Update the accepted-finding, false-positive, and related-issue summaries before the next recheck.

Keep main-agent source reads minimal. If a candidate needs broad source reading,
send a targeted recheck subagent instead of expanding the main context.
Also compare candidate reports for meta-problems in rules or design docs, not
just code defects.

# Audit Prompt Templates

Use these templates with mini subagents. Keep prompts scoped and self-contained.

## Initial Source Slice

```text
You are a lightweight source-audit subagent for tensor4all-rs.

Scope:
- Repository: /home/shinaoka/tensor4all/tensor4all-rs
- Files/directories: <exact list>
- Rules to check: README.md, REPOSITORY_RULES.md, AGENTS.md excerpts below.

Task:
Inspect only your assigned scope for possible repository-rule violations, correctness bugs, missing path coverage, stale docs/examples, hidden dense materialization, index identity mistakes, C API error-loss patterns, public API drift, or unsafe layering.

Output possible issues only. Do not edit files. Do not create issues. Do not assign final severity.

For each candidate:
- id: local candidate id
- file:line
- rule: violated rule or invariant
- evidence: concrete source/API-doc evidence with a short snippet or precise line summary
- impact: likely consequence if real
- confidence: low/medium/high
- related_search: suggested rg query or related files for the main agent

If none, say "No possible issues found in assigned scope" and list the files you inspected.
```

## Targeted Recheck

```text
You are a lightweight recheck subagent for tensor4all-rs.

Scope:
- Repository: /home/shinaoka/tensor4all/tensor4all-rs
- Files/directories or rg queries: <exact list>

Known accepted findings so far:
<paste short known-issues summary only, not the full issue draft>

Task:
Look for new possible issues, broader affected paths, or evidence that a known finding is duplicated elsewhere. Do not repeat known findings unless you add new evidence, broader scope, or a distinct root cause.
Preserve all known-finding paths, APIs, and evidence exactly as provided. If you inspect additional likely-related paths, list them as recheck scope or new evidence, not as rewritten known facts.

Output possible issues only. Do not edit files. Do not create issues. Do not assign final severity.

For each candidate:
- id
- relation: new-root-cause | broader-scope | possible-duplicate | refutation
- file:line
- evidence: short snippet or precise line summary
- impact
- confidence
- related_search
```

## Main-Agent Triage Checklist

For every candidate:

1. Open the cited source/API docs and verify the claim.
2. Check whether the claim is a duplicate of an accepted root cause.
3. Search related files with the candidate's `related_search` or a better `rg` query.
4. Decide final severity and record the rationale.
5. Update the known-issues summary before the next recheck.

Keep main-agent source reads minimal. If a candidate needs broad source reading,
send a targeted recheck subagent instead of expanding the main context.

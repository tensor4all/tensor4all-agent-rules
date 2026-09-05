---
name: tensor4all-rules-audit
description: Audit a tensor4all repository's applicable rules and source for concrete problems, duplication and excessive process, returning a bounded evidence-based report.
---

# Tensor4all Rules Audit

Audit directly in the main agent. Subagents and cross-model reviews are optional
and require an explicit user request; they are not prerequisites for an audit.
An audit request does not by itself authorize implementation, issues or PRs.

## Workflow

1. **Bound the scope.** Identify the requested repository, working state and rule
   files. Read applicable rules and referenced decisions, not the entire workspace
   by default. A local or uncommitted target is valid; describe it accurately.
   Do not require a push or a GitHub-visible commit to inspect local work.
2. **Inspect the real path.** Read enough source to understand the contract,
   callers and existing checks. Use existing API documentation when helpful;
   do not build release-mode documentation as a default audit prerequisite.
3. **Verify findings.** Require a precise location, evidence, impact and smallest
   useful correction. Distinguish confirmed defects, source risks and policy/docs
   gaps. A rule violation alone does not prove a severe runtime bug.
4. **Check nearby instances once.** Group a confirmed problem by root cause and
   check the relevant neighboring paths. Do not iterate broad scans until no
   model can invent another finding. Report unresolved scope honestly.
5. **Report and stop.** Provide a ranked, concise list of findings, inspected
   scope and limitations. Reuse existing issues when known. Create an issue or
   implement fixes only when requested; do not automatically turn an audit into
   an open-ended remediation program.

## Rule Audits

For each process rule, ask:
- What concrete failure does it prevent?
- Is it already enforced by CI, tools or another rule?
- Is its scope appropriate for a small fix?
- Does it force delegation, repeated approvals, duplicate records or unnecessary
  build/network work?

Prefer deleting redundant instructions or adding a clear applicability condition
rather than inventing another checklist. Preserve actual numerical, memory,
aliasing, device, data-loss and publication safety requirements.

## Evidence and Review

- Verify findings against actual source or a reproducer. Do not add code to
  appease an unsupported review claim.
- Use repository-relative paths and the inspected revision/state. Add stable
  permalinks when publishing references to an available remote commit.
- After a fix, rerun the relevant check and inspect its delta; do not restart a
  design approval or full audit unless the scope materially changed.
- If an independent opinion is explicitly requested, use the optional
  [bounded prompt](references/audit-prompts.md). Its output is evidence to assess,
  not an automatic approval gate.

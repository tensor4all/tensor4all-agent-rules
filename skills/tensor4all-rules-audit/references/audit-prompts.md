# Optional Independent Review Prompt

Use only when the user explicitly requests an independent opinion. No particular
model family, number of reviewers, or follow-up round is mandatory.

```text
Review only <files/diff and revision or working state>, read-only.
Question: <specific correctness or policy concern>.
Relevant contract: <short rule or design reference>.
Known evidence: <tests/source facts, including rejected claims if relevant>.

Read actual source rather than relying on summaries. Return concrete findings
with locations, short evidence and impact. Separate confirmed defects from
uncertainty and policy gaps. State what was not inspected. Do not propose new
frameworks, broaden the audit or create issues. No findings is a valid outcome.
```

The main agent assesses the evidence. An inaccurate claim is rejected with a
source explanation; it does not require a code change or another review round.

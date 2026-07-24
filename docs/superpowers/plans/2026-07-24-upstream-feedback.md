# Upstream Feedback Rule Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a shared agent rule that recommends reporting bugs to relevant upstream libraries while requiring separate user approvals before drafting and before submission.

**Architecture:** Extend the existing provenance policy because it already governs referenced and ported-from upstream work. Validate the wording with a focused content assertion and the repository's existing rules-index link check, then merge the completed branch into local `main`.

**Tech Stack:** Markdown, Python 3 standard library, Git

## Global Constraints

- Apply the rule to libraries used as references, implementation guides, or sources of ports.
- Recommend an upstream issue or pull request; do not start one automatically.
- Require explicit user permission before preparing an upstream-facing issue draft or pull-request patch.
- Require separate explicit user permission immediately before external submission.
- Permission to draft must never imply permission to submit.
- Do not change authorization for repository-local investigation or requested local fixes.

---

### Task 1: Add and validate the upstream feedback rule

**Files:**
- Modify: `rules/common/provenance.md`
- Modify: `rules/index.md`

**Interfaces:**
- Consumes: the provenance policy linked from `rules/index.md`
- Produces: a new `Return Upstream Bug Findings` policy section

- [ ] **Step 1: Run a focused check that demonstrates the rule is absent**

Run:

```bash
python3 - <<'PY'
from pathlib import Path

text = Path("rules/common/provenance.md").read_text(encoding="utf-8")
normalized = " ".join(text.split())
required = [
    "## Return Upstream Bug Findings",
    "before preparing an upstream-facing issue draft or pull-request patch",
    "separate explicit permission immediately before creating the issue or pull request",
    "Permission to prepare a draft does not authorize external submission.",
]
missing = [fragment for fragment in required if fragment not in normalized]
assert not missing, f"missing upstream-feedback policy fragments: {missing}"
PY
```

Expected: FAIL with `missing upstream-feedback policy fragments`.

- [ ] **Step 2: Add the minimal policy section**

Append this section to `rules/common/provenance.md`:

```markdown
## Return Upstream Bug Findings

- When work reveals a likely bug in a library used as a reference,
  implementation guide, or source of a port, report the finding and supporting
  evidence to the user. Recommend giving the finding back to upstream as an
  issue or pull request.
- Ask for explicit user permission before preparing an upstream-facing issue
  draft or pull-request patch. If permission is not given, do not begin that
  upstream-facing draft or patch.
- Show the completed draft or patch to the user, then ask for separate explicit
  permission immediately before creating the issue or pull request upstream.
  Permission to prepare a draft does not authorize external submission.
- These approval requirements govern upstream-facing preparation and
  submission. They do not by themselves restrict repository-local
  investigation or fixes already requested by the user.
```

- [ ] **Step 3: Re-run the focused policy check**

Run the Python command from Step 1.

Expected: PASS with exit status 0 and no output.

- [ ] **Step 4: Advertise the policy in the rules index**

Extend the `common/provenance.md` description in `rules/index.md` to include:

```text
permission-gated upstream bug feedback
```

Run:

```bash
python3 - <<'PY'
from pathlib import Path

text = " ".join(Path("rules/index.md").read_text(encoding="utf-8").split())
required = "permission-gated upstream bug feedback"
assert required in text, f"rules index does not advertise: {required}"
PY
```

Expected: PASS with exit status 0 and no output.

- [ ] **Step 5: Run repository validation**

Run:

```bash
python3 - <<'PY'
import pathlib
import re

rules = pathlib.Path("rules")
text = (rules / "index.md").read_text(encoding="utf-8")
links = re.findall(r"\]\(([^)#]+\.md)[^)]*\)", text)
missing = sorted({link for link in links if not (rules / link).exists()})
assert not missing, f"broken links in rules/index.md: {missing}"
print(f"rules/index.md OK: {len(links)} relative links resolve")
PY

git diff --check
```

Expected: `rules/index.md OK: 10 relative links resolve`; `git diff --check` exits successfully.

- [ ] **Step 6: Commit the rule**

```bash
git add rules/common/provenance.md rules/index.md
git commit -m "rules: require approval for upstream feedback"
```

Expected: one commit containing the provenance policy and its index description.

### Task 2: Verify and merge the completed branch

**Files:**
- Verify: `docs/superpowers/specs/2026-07-24-upstream-feedback-design.md`
- Verify: `docs/superpowers/plans/2026-07-24-upstream-feedback.md`
- Verify: `rules/common/provenance.md`
- Verify: `rules/index.md`

**Interfaces:**
- Consumes: the validated `provenance-rules` branch
- Produces: local `main` containing the completed provenance and upstream-feedback rules

- [ ] **Step 1: Run final branch verification**

Run the focused policy check and repository validation from Task 1, then:

```bash
git diff --check main...HEAD
git status --short
```

Expected: all checks pass; only the locally generated `.codegraph/` directory may remain untracked.

- [ ] **Step 2: Confirm the commits to merge**

Run:

```bash
git log --oneline --decorate main..HEAD
```

Expected: only the provenance-rule work, its design and plan, and the upstream-feedback rule commit are listed.

- [ ] **Step 3: Merge into local main**

```bash
git switch main
git merge --ff-only provenance-rules
```

Expected: local `main` advances by fast-forward with no conflicts.

- [ ] **Step 4: Verify the merged result**

Run the focused policy check and repository validation from Task 1, then:

```bash
git status --short --branch
git log -4 --oneline --decorate
```

Expected: validation passes and `main` points to the upstream-feedback rule commit; `.codegraph/` may remain untracked and is not committed.

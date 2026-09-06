# Agents As Consumers Of Tensor4all Libraries

The other rule files address agents that change tensor4all code. This file
addresses work that serves humans and coding agents who discover a tensor4all
library, learn it cold, call it correctly on the first try, and self-correct
from its errors. Agent-facing surfaces are exercised far more often than human
documentation, so misleading ones fail far more often.

Each policy names the check that guards it. A policy without a failing check is
a suggestion; land the check in the same change as the policy.

## Ship A Downstream-Usage Skill

- Every user-facing library repository ships a downstream-usage skill in the
  Agent Skills format (`SKILL.md` with YAML frontmatter plus a `references/`
  directory) covering: which package or crate to depend on for which task;
  imports or preludes needed before the first call; layout and indexing
  conventions that produce silently wrong answers when guessed (memory order,
  index base, tolerance semantics); performance idioms (what to construct once
  and reuse, what to compile once and run many times); known first-try failure
  modes with fixes.
- The skill addresses library users. Contribution workflows (issue intake, PR
  conventions, release steps) belong in separate skills.
- Bundle the skill in the repository and link it from the README with one
  sentence saying when to load it. The README is the registry landing page, so
  the skill is reachable from crates.io or the registry without external
  distribution.
- Compilable snippets in the skill go through the repository's snippet
  verification, the same one guarding guides and tutorials. A stale skill is
  worse than none: agents copy it without skepticism.
- Guard: the pre-PR checklist requires that a change to public API surface,
  feature flags, package boundaries, or documented idioms reviews the shipped
  skills and updates them in the same PR, mirroring design-document freshness.

## Name The Remedy In Error Messages

- When a documented alternative API, explicit conversion, feature flag, or
  supported-value set provides one reliable remediation, the message appends it
  as `<what failed>; <what to do>`.
- Do not invent a remedy when no universal next action exists; a precise
  parameterized diagnosis suffices for open-ended failures.
- The error string is an agent's primary self-correction input: a remedy clause
  makes a one-step fix, a bare diagnosis sends it to guesswork.
- Guard: repository error-message rules cite this policy, and review of new or
  changed error variants checks for an applicable remedy.

## Require Package-Index Metadata Before Publication

- Before publication, the manifest carries the fields that make the package
  discoverable and fully documented on the index: keywords and categories (or
  registry equivalents), minimum supported language version, and
  documentation-build configuration.
- For Rust, docs.rs builds default features only; feature-gated public API is
  invisible without `[package.metadata.docs.rs]` (`all-features = true`, or an
  explicit feature list when features are mutually exclusive). An agent reading
  hosted docs concludes an undocumented feature-gated API does not exist. Julia
  packages apply the General registry and docs-hosting equivalents.
- Guard: a pre-publish layout check fails when these fields are missing,
  rather than relying on a checklist read at release time.

## Publish A Verified llms.txt Index

- A user-facing documentation site publishes `llms.txt` at its root: a Markdown
  list of the authoritative pages with one-line descriptions, so an agent
  replaces crawling with one fetch. Include the downstream-usage skill.
- Prefer generating the list from the site manifest. If hand-maintained, a
  check verifies every listed page exists and every authoritative page is
  listed.
- Guard: the docs-site check resolves every `llms.txt` entry and fails the
  build on a broken or missing one.

## Validate From The Consumer's Seat

- The proof these surfaces work is the source-blind build test in
  [`docs-and-tests.md`](docs-and-tests.md): a doc-only agent writes a minimal
  integration from the published docs and skill alone, compile-checked against
  the real project. Run it when introducing or substantially revising a usage
  skill, quickstart, or llms.txt index.

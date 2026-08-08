# Agents As Consumers Of Tensor4all Libraries

The other rule files address agents that change tensor4all code. This file
addresses repository work that serves the opposite side: humans and coding
agents that discover a tensor4all library, learn it cold, call it correctly on
the first try, and self-correct from its errors. These rules exist because
agents amplify both directions: good agent-facing surfaces are exercised far
more often than human documentation, and misleading ones fail far more often.

Each policy below names the check that guards it. A policy without a failing
check is a suggestion; prefer landing the check in the same change as the
policy.

## Ship A Downstream-Usage Skill

- Every user-facing library repository ships a downstream-usage skill in the
  Agent Skills format (a `SKILL.md` with YAML frontmatter plus a `references/`
  directory for detail files), covering: which package or crate to depend on
  for which task, the imports or preludes needed before the first call, the
  layout and indexing conventions that produce silently wrong answers when
  guessed (memory order, index base, tolerance semantics), performance idioms
  (what to construct once and reuse, what to compile once and run many times),
  and known first-try failure modes with their fixes.
- The skill addresses users of the library. Contribution workflows (issue
  intake, PR conventions, release steps) belong in separate skills.
- Bundle the skill in the repository and link it from the README with one
  sentence saying when to load it. The README is the package registry landing
  page, so an in-repo skill linked from the README is reachable from
  crates.io or the registry without any external distribution.
- Compilable snippets inside the skill go through the repository's snippet
  verification mechanism, the same one that guards guides and tutorials. A
  stale skill is worse than none: agents copy it without skepticism.
- Guard: the repository's pre-PR checklist requires that a change to public
  API surface, feature flags, package boundaries, or documented idioms
  reviews the shipped skills and updates them in the same PR, mirroring the
  design-document freshness rule.

## Name The Remedy In Error Messages

- When a documented alternative API, explicit conversion, feature flag, or
  supported-value set provides one reliable remediation, the error message
  appends it to the diagnosis as `<what failed>; <what to do>`.
- Do not invent a remedy when no universal next action exists; a precise
  parameterized diagnosis is correct on its own for genuinely open-ended
  failures.
- The error string is the primary self-correction input for an agent. A
  remedy clause turns a failed run into a one-step fix; a bare diagnosis
  sends the agent to training-data guesswork.
- Guard: repository error-message rules cite this policy, and review of new
  or changed error variants checks for an applicable remedy.

## Require Package-Index Metadata Before Publication

- Before a package is published, its manifest carries the fields that make it
  discoverable and fully documented on the package index: keywords and
  categories (or the registry's equivalents), the minimum supported language
  version, and documentation-build configuration.
- For Rust, docs.rs builds default features only; feature-gated public API is
  invisible without `[package.metadata.docs.rs]` (`all-features = true`, or
  an explicit feature list when features are mutually exclusive). An agent
  that reads the hosted docs concludes an undocumented feature-gated API does
  not exist. Julia packages apply the General registry and docs-hosting
  equivalents.
- Guard: a pre-publish layout check fails when these fields are missing,
  rather than relying on a checklist read at release time.

## Publish A Verified llms.txt Index

- A user-facing documentation site publishes `llms.txt` at its root: a
  Markdown list of the authoritative pages, each with a one-line description,
  so an agent can replace crawling with a single fetch. Include the
  downstream-usage skill in the list.
- Prefer generating the page list from the site manifest. Where it is
  maintained by hand, a check must verify that every listed page exists and
  every authoritative page is listed, so the index cannot drift silently.
- Guard: the docs-site check resolves every `llms.txt` entry and fails the
  build on a broken or missing entry.

## Validate From The Consumer's Seat

- The proof that these surfaces work is the source-blind build test defined
  in [`docs-and-tests.md`](docs-and-tests.md): a doc-only agent writes a
  minimal integration from the published docs and skill alone, and that code
  is compile-checked against the real project. Run it when introducing or
  substantially revising a usage skill, quickstart, or llms.txt index.

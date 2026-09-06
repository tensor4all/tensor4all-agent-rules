# Provenance, Copyright, And Scientific Credit

These rules apply whenever code is written while referencing third-party code,
designs, or algorithms, by a human or an AI agent. Record provenance at the
moment of writing; it is far harder to reconstruct later.

## Record References In The Source

- When code is generated or written with a third-party implementation as a
  reference, leave a file- or module-level comment naming the project, file,
  and where useful the line range or function.
- Do this at writing time.
- State the nature of the reference when not obvious: ported from, derived
  from, following the conventions of, or validated against.

## Comply With Copyright

- Copyright protects expression, not algorithms. Implementing a published
  algorithm independently carries no license obligation; translating code does.
- Close translation (writing while reading the upstream source, function by
  function or line by line) is a derived work even across languages. It
  retains the original copyright and license: keep a derivation notice in the
  affected file, include the upstream license text in the repository or crate,
  and set the package license metadata to the correct SPDX expression (e.g.
  `MIT AND Apache-2.0`).
- Do not claim independence the code lacks. "No source code was copied" must
  be literally true; if the code follows an upstream structure, say so.

## Respect Scientific Credit Beyond The License

- Upstream projects and the original papers of implemented algorithms deserve
  visible credit even without a copyright obligation.
- When work introduces or reveals such a relationship, propose an entry in the
  repository's Provenance and Citation Policy (or equivalent, such as an
  acknowledgments section) and add it **after confirming with the user**. The
  user decides how credit is expressed; the agent notices and asks.
- Follow the repository's citation policy style where one exists: cite original
  algorithm papers, cite upstream library papers, and apply upstream citation
  policies recursively.

## Return Upstream Bug Findings

- When work reveals a likely bug in a library used as a reference, guide, or
  port source, report the finding and evidence to the user and recommend giving
  it back upstream as an issue or PR.
- Ask for explicit user permission before preparing an upstream-facing issue
  draft or patch; without it, do not begin.
- Show the completed draft or patch to the user, then ask for separate explicit
  permission immediately before creating the upstream issue or PR. Permission
  to draft does not authorize submission.
- These approvals govern upstream-facing preparation and submission only; they
  do not restrict repository-local investigation or fixes the user already
  requested.

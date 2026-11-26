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

## AI Disclosure On GitHub Issues


- Every AI-written GitHub Issue body and Issue comment must visibly disclose
  that it was written by AI and name the model that wrote it. This applies
  whether or not a human requested, reviewed, or approved the publication.
- Put the disclosure in the published body itself, not only in hidden HTML
  metadata, local logs, or a separate follow-up comment. Preserve it when editing.
- Use the actual model name supplied by the execution environment. Do not infer
  a model from the account name or copy the model from an example. If the exact
  model identifier is unavailable, disclose that explicitly rather than invent it.
- Distinguish a human request or approval to publish from human review of the
  contents. Do not claim human review or endorsement unless it actually occurred.
- For automated publication without human approval, include a visible
  **Provenance** footer stating the agent and account, the absence of a human
  request for that specific post, the actual automation trigger/action and target
  link, the model, and the execution timestamp with timezone. Include
  "Review it before relying on it." If there was a human request but no review
  of the resulting text, state that accurately instead of claiming no request.
- These disclosure requirements do not themselves authorize automated posting.

Example for a human-requested post (replace the model placeholder):

> **Provenance** — Written by AI (model: `<actual model name>`) at Hiroshi's request.

Example for an automated post (the details below are an example, not defaults):

> **Provenance** — written by the `tensor4all` agent on Hiroshi's account,
> **not** on a human request: scheduled issue/PR sweep that answered
> [#2563](https://github.com/tensor4all/tenferro-rs/issues/2563).
> Model `deepseek-v4.1-flash`, 2026-11-10 17:26 JST. Review it before relying on it.

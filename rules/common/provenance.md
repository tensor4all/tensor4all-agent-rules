# Provenance, Copyright, And Scientific Credit

These rules apply whenever code is written while referencing third-party
code, designs, or algorithms, whether by a human or by an AI agent. They
exist because AI-assisted development makes writing one codebase while
referencing another effortless, so provenance must be recorded at the moment
of writing, not reconstructed later.

## Record References In The Source

- When code is generated or written with a third-party implementation as a
  reference, leave a reference to the source in the code: project name, file,
  and where useful the line range or function name, in a file-level or
  module-level comment.
- Do this at writing time. A missing provenance comment is much harder to
  recover after the fact than to write in the moment.
- Distinguish the nature of the reference in the comment when it is not
  obvious: ported from, derived from, following the conventions of, or
  validated against.

## Comply With Copyright

- Copyright protects expression, not algorithms. Implementing a published
  algorithm independently carries no license obligation; translating code
  does.
- Close translation (writing while reading the upstream source, function by
  function or line by line) produces a derived work even across languages.
  Derived code retains the original copyright and license: keep a derivation
  notice in the affected file, include the upstream license text in the
  repository or crate, and set the package license metadata to the correct
  SPDX expression (for example `MIT AND Apache-2.0`).
- Do not claim independence the code does not have. Statements such as "no
  source code was copied" must be literally true; if the code follows the
  structure of an upstream implementation, say so instead.

## Respect Scientific Credit Beyond The License

- Even when there is no copyright obligation, upstream projects and the
  original papers of implemented algorithms deserve visible credit.
- When work introduces or reveals such a relationship, propose an entry in
  the repository's Provenance and Citation Policy document (or its
  equivalent, such as an acknowledgments section), and add it **after
  confirming with the user**. The user decides how credit is expressed;
  the agent's job is to notice that credit is due and to ask.
- Follow the affected repository's citation policy style where one exists:
  cite original algorithm papers, cite upstream library papers, and apply
  upstream citation policies recursively.

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

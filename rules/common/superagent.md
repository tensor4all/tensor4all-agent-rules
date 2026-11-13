# Common Superagent Rules

A superagent is a resident agent that patrols repositories, dispatches workers,
and reviews or merges pull requests without a human in each step. These rules
add what is specific to that role. [`assignment.md`](assignment.md) and
[`provenance.md`](provenance.md) still apply in full; the setup itself is
described in [`docs/agent-fleet`](../../docs/agent-fleet/README.md).

Every rule below also binds an interactive agent when it opens, reviews, holds,
or merges a pull request. The superagent is only the actor that does it most.

## Dispatch Only `ai-ready`, Never `human-approve-required`

- Dispatch only open, unassigned issues labelled `ai-ready`. A `bug` label, an
  issue body that reads as finished design, or the issue's age is not
  readiness.
- Labelling is the maintainer's superagent's job, applied while it patrols. No
  label means no dispatch.
- `human-approve-required` marks an issue or PR that needs an explicit human
  decision (public API, accepted design, release, new dependency, governance).
  Never dispatch it, never merge it, and never remove the label yourself; a
  human removes it when they decide. If it appears on an in-flight PR,
  convert the PR to draft.
- An issue opened minutes ago by someone else is often backed by their local
  fix. Leave it alone unless it is `ai-ready` and unassigned.

## Every PR Names Its Model

- An agent-authored PR body carries one line-initial
  `Model: <provider>/<model>` line per model that wrote code, for example
  `Model: anthropic/claude-opus-5-5`. A PR without it is converted to draft by
  the patrol and is not merged.

## Review By A Different Provider Is Required To Merge

- Before merge, a model from a **different provider** than every `Model:` line
  reviews the PR. Two models of one provider (for example two Claude models)
  do not count.
- The review comment names its `Model: <provider>/<model>` and the head SHA it
  read. The reviewing agent then adds the `xreview:done` label itself.
- A push after that SHA voids the review: remove `xreview:done` and review the
  delta again.
- A review that did not read the whole diff (truncated input, skipped files)
  says so and does not earn `xreview:done`. Split the PR or review the rest.
- Do not enable auto-merge until `xreview:done` is present.
- An author without access to a second provider adds `xreview:requested`. A
  maintainer's superagent picks it up, chooses a provider different from the
  author's, and reads the diff without building or running PR code from
  authors outside the organization.
- This is an agreement, not an enforced gate. The label exists so that the
  agent has to take the step consciously and leaves a record of it.

## A Hold Is A Draft

- To stop a merge, convert the PR to draft in the same action as the change
  request or the "wait before merging" comment:
  `gh pr ready <n> --undo`, plus one line that says `hold:` and why.
- Measured on GitHub (2026-11-14): converting to draft disables auto-merge,
  auto-merge cannot be enabled on a draft, and marking the PR ready again does
  not re-enable it. A hold that lives only in a comment, a journal or a
  `CHANGES_REQUESTED` state does not stop auto-merge
  (tensor4all-rs#1110 merged 57 s after a change request; strided-rs#435
  merged under a hold recorded only in a journal).
- Only the agent or person who set the hold, or a maintainer, marks the PR
  ready again. A green CI is not a release. Re-enabling auto-merge is a
  separate, explicit step.

## Re-read Before Consequential Actions

- Immediately before merging, closing, reopening, or handing over a PR, re-read
  its state, head SHA, draft flag, labels, and comments newer than the last
  review, and the state of any PR you are about to call a duplicate.
- Do not merge while a review or an unanswered steering comment is newer than
  the last review (strided-rs#451 and tensor4all-rs#1223 merged 30-51 s before
  a finding was posted).
- Do not act on an event or a snapshot from an earlier tick
  (tensor4all-rs#1230 was closed as a duplicate of a PR that had been closed
  15 minutes earlier).

# Common Assignment And Ownership Rules

An issue's assignee list is the lock that says who is working on it. It is the
only signal that survives across sessions, machines, and agent fleets, so treat
it as authoritative for both sides of the question: who may start, and who must
stay away.

## Claim Before You Start

- **Assign yourself before the first read that matters.** `gh issue edit <n>
  --add-assignee @me` on an unassigned issue is the whole ceremony; there is no
  approval step and no reason to ask first.
- Read the current assignees first, not the issue body. An issue whose body
  looks unstarted may already be held by a parallel session.
- **Confirm the claim landed.** `gh api repos/<owner>/<repo>/issues/<n>` returns
  exit status 0 with a `{"message":"Not Found"}` body for a bad number, so a
  parser that trusts the exit status records "held by someone else" for an issue
  that does not exist. Reject anything containing `{`, `}` or `:` before treating
  the output as a list, then read the field back.
- If the read-back does not show you, stop. A claim that did not stick is a
  claim that will be stolen by the next scheduled sweep.

## Do Not Take Another Person's Issue

- **An issue assigned to someone else is theirs.** Do not dispatch a worker to
  it, do not push a branch for it, and do not "help" by starting the obvious part.
  Comment, or ask, and move on to unassigned work.
- This holds for agents and maintainers who joined recently: their assignment
  is not a placeholder. Hand the work back rather than taking it over.
- When an issue must move between people, the outgoing assignee (or a
  maintainer) changes the assignment explicitly; silence is not a handover.

## Check Again Before You Open A PR

Ownership can move while you work, so the check at the start is not the check
that counts:

- **Re-read the assignees immediately before opening the pull request.** If the
  issue is now assigned to someone else, stop and ask instead of opening.
- **Re-read the open pull requests too.** A branch whose name ends in `-<n>`, a
  title containing `#<n>`, or a body containing `Closes #<n>` or `Fixes #<n>` is
  already an implementation of that issue, even when it was opened after you
  started. Two live pull requests for one issue is a merge conflict between
  people, not a technical conflict.
- When a duplicate does appear, the decision to close one belongs to whoever
  owns the pull request. Read its state and its author first; the author may
  already have closed the other one.

## Automation

The rule is cheap enough to make mechanical, and it should be, because a
scheduled sweep has no memory of a conversation:

- Wrap the start of work and the opening of a pull request in one script that
  reads the assignees, claims if empty, refuses if the issue belongs to someone
  else, and verifies the result. Two entry points that both call it are better
  than two places that both remember a rule.
- Exit codes should distinguish the outcomes, so the caller can tell "not mine"
  from "the claim failed" and from "the issue does not exist".
- Test the guard on both directions: an issue you hold, and one you do not.
  A guard that only ever runs on unassigned issues has not been tested.

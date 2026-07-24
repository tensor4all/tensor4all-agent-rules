# Upstream Feedback Rule Design

## Goal

Encourage agents to return useful bug findings to upstream libraries that were
used as references, consulted during implementation, or served as the source
of a port, while keeping all upstream-facing work under explicit user control.

## Placement

Add the rule to `rules/common/provenance.md`. The existing file already governs
how agents handle relationships with referenced and ported-from upstream work,
so it is a better fit than the general repository workflow rules or a new
standalone rule file.

## Required Behavior

When an agent identifies a likely bug in a relevant upstream library, it
should:

1. Report the finding and supporting evidence to the user.
2. Recommend giving the finding back to upstream as an issue or pull request.
3. Ask for explicit permission before starting an upstream-facing issue draft
   or pull-request patch.
4. Show the completed draft or patch to the user.
5. Ask for separate explicit permission immediately before creating the issue
   or pull request upstream.

Permission to prepare a draft does not authorize external submission. If
permission is not given at either stage, the agent must stop that upstream
feedback activity at the corresponding boundary.

## Scope

This rule applies to upstream libraries used as a reference, consulted as an
implementation guide, or used as the source of a port. It governs preparation
and submission of upstream issues and pull requests; it does not by itself
change authorization for repository-local investigation or fixes requested by
the user.

## Validation

Review the final wording to ensure that:

- upstream feedback is recommended rather than performed automatically;
- permission is required before drafting;
- a second permission is required before submission; and
- draft permission cannot be interpreted as submission permission.

---
name: dependency-release-chain
description: Raise a tensor4all repository's pins to the latest published versions of the stack it depends on (strided → tcontract → cpueinsum/tlinalg → tenferro), including the re-releases that a pinned middle makes necessary. Use when asked to bump dependencies to the latest, to consume an upstream release, to re-release a stack, or when a pin bump fails to resolve.
---

# Dependency release chain

The stack pins exact versions (`=X.Y.Z`) on purpose: one registry source per crate, so
the shared `StridedView` and the executor handle types meet at a crate boundary. That
policy is what turns an upstream patch into a *chain* rather than an edit.

## 0. Decide the scope first, in words

"Raise the dependency" and "release this repository" are different tasks with different
outputs. State which one you are doing, and stop if the answer changes the work: a
dependency move does **not** bump the consumer's own version and does not publish it.

## 1. Audit before editing

For every pin, compare it with the registry's latest stable
(`crates.io/api/v1/crates/<name>` → `max_stable_version`). Report the ones that lag and
leave the ones that do not — often the answer is "one crate", not "the stack".

## 2. When the consumer cannot take the new version, read the resolution error

An exact-pin chain resolves only if every crate in it accepts the target. A failing
`cargo metadata` is the signal, and its message names the *published* requirement that
blocks you (for example `tcontract 0.2.0` requiring `strided-basic =0.5.0`). Never guess
which crate is in the way: read it, then re-release that crate first.

## 3. Re-release the middle, in dependency order

- Bump the pin, the crate's own version and every internal cross-crate requirement in the
  same change; a missed one is a resolution failure, not a warning.
- Run that repository's own gate from its `AGENTS.md` — this skill does not restate it.
- Merge, then tag the **merged** commit, pushed and reachable from `main`.
- Publish from a clean, detached worktree at that tag, in dependency order (`kernel` →
  `exec` → `contract`; `cpueinsum` → `cpueinsum-blas`).
- **Verify the published registry requirements**, not the local manifest: `cargo publish`
  strips `git` sources and keeps only `version`, so
  `/api/v1/crates/<name>/<version>/dependencies` is the evidence that the chain moved.

## 4. Publish only with approval

An agent may publish once the maintainer approves **this release**, in the conversation.
A package that is not already on the registry needs its own approval naming it, asked for
each such package. Publication is irreversible, and the rest of the release safety rules
in `rules/common/repository.md` apply unchanged.

## 5. Move the consumer's pins, and nothing else

- `cargo metadata` must resolve and `cargo tree` must show one crate of each; state the
  resolved chain in one line (`A → B → C`).
- Its contract tests may encode assumptions about the pins — for example that a family
  carries one identical version string. Fix the test's *intent*, not its teeth: a family
  that publishes independent patches needs "one release line", not "one string".
- Its gate scripts may fetch `origin`; where the remote is SSH-only and unusable in this
  environment, switch it to HTTPS for the gate and restore it afterwards.
- No version bump and no release for the consumer when the task was the dependency move.

## 6. Report

The chain that now resolves, what was re-released, what you deliberately did not touch and
why, the gate output you actually ran, and what the registry says you have not checked.

# Rust Build Artifacts And Worktree Cleanup

Applies to any Rust crate or workspace, with or without a compiler cache.

## Where Build Outputs Live

- Cargo writes every crate's artifacts into the **consumer's** `target/`, never
  into the dependency's source directory. A shared git dependency compiled by
  fifteen clones produces fifteen target-tree copies (cheap only when the
  filesystem shares extents between them).
- `~/.cargo/git/checkouts/**` and `~/.cargo/registry/**` hold **sources only**.
  They are re-fetchable, contain no compiler artifacts, and are not cleanup
  targets.
- When a compiler cache wraps `rustc` (`RUSTC_WRAPPER`, for example kache or
  sccache), there is a second copy: one global store on the machine
  (`~/.cache/kache/store` plus `index.db`) shared by every project, worktree,
  and unrelated repository. Cache entries are commonly materialized as
  reflinks or hardlinks inside build trees, so the same bytes are visible in
  both places; `kache stats` reports them as `private` and `cloned into
  target/`.
- Building an upstream repository itself (a checkout or worktree of it) puts its
  artifacts in **that** tree's `target/`. Deleting those is a full rebuild of
  that repository and cannot be recovered from a consumer's tree.

## Locate Before Deleting

- Enumerate real Cargo target dirs, not names: require `target/.rustc_info.json`
  (a source directory may legitimately be called `target`).
- Respect custom locations: `CARGO_TARGET_DIR` and `build.target-dir` in
  `.cargo/config.toml` move artifacts outside `<repo>/target`. Find them instead
  of assuming the default path.
- Treat `target/` as derived data and the worktree, source tree, `.git`, and
  uncommitted files as off limits. Only `target/` is ever deleted.

## Never Delete While A Build Is Running

- Check for live builds before touching anything: for each `cargo` and `rustc`
  process, resolve `/proc/<pid>/cwd` and skip any tree that is a prefix of it.
  Check `lsof`/`fuser` on `target/.cargo-lock` when available; the lock file's
  presence alone is not a signal, since Cargo leaves it behind.
- Deleting a target under a running build breaks that build and can strand the
  cache entry it was writing.

## Preserve By Default

Default cleanup scope is **the worktree currently being worked in**, or paths the
user names explicitly. Do not sweep a home directory or a shared parent by
default, and never delete these without an explicit request:

- Other repositories, and other worktrees or worktree farms of any repository
  (for example a sibling agent session's `.worktrees/<task>` tree).
- Upstream checkouts and worktree farms kept for review, such as a
  `<upstream>-rs` worktree used to work on that dependency.
- `~/.cargo/git/checkouts/**` and `~/.cargo/registry/**`.
- Any target tree whose build is in flight.

Rationale: keeping a worktree's `target/` **is** the cheapest cache. Deleting it
forces a rebuild of that tree's dependency graph, which a compiler cache only
softens when it still holds a private copy of those entries.

## Order Of Operations For A Safe Sweep

1. Scope: the current worktree (or explicit `--root` paths).
2. Skip trees with live builds, trees modified inside the retention window, and
   anything on the preserve list.
3. Delete only verified `target/` directories, oldest first.
4. Converge the cache: run the cache's garbage collector (`kache gc`) so the
   index drops entries whose data no longer exists, then confirm the health
   counters (`kache stats`: store within `local_max_size`, `entries_unreclaimable`
   at or near zero). Skipping this step leaves metadata for vanished data and
   makes the reported store size meaningless.
5. Report what was deleted, what was preserved and why, the bytes freed, and the
   post-GC store state.

A compact, dry-run-first sweep:

```bash
is_live() {  # $1 = tree; Cargo's cwd is the workspace root, so prefix-match it
  for p in $(pgrep -x cargo; pgrep -x rustc); do
    cwd=$(readlink "/proc/$p/cwd" 2>/dev/null || true)
    case "$cwd" in "$1"|"$1"/*) return 0;; esac
  done
  return 1
}
find "$root" -type d -name target -prune 2>/dev/null | while read -r d; do
  [ -f "$d/.rustc_info.json" ] || continue
  tree="${d%/target}"
  is_live "$tree" && { echo "SKIP in-use $d"; continue; }
  [ -z "$(find "$d" -maxdepth 0 -mtime +"${DAYS:-7}")" ] || { echo "DELETE $d"; [ "${APPLY:-0}" = 1 ] && rm -rf "$d"; }
done
[ "${APPLY:-0}" = 1 ] && kache gc
```

## Prefer Cache Trimming Over Deleting Other People's Trees

- Measure before choosing a target. `kache list --json` aggregated by crate and
  hit count shows how much of the store was ever reused; a store dominated by
  `hits == 0` entries can lose most of its bytes without losing reuse, because
  an entry with no private copy cannot serve a hit at all.
- Lower `local_max_size` (for example to a few GiB, sized to the hot set) and run
  `kache gc` to reclaim store space without touching any worktree. This is the
  only lever that frees disk without costing someone a rebuild.
- Use `kache gc --max-age` and `kache gc --stale-schema` for one-off trims, and
  `kache purge --crate-name <crate>` for a single offender.
- Never `kache purge` or delete the cache store to free disk: the private share
  is usually small, and the reuse value is real. Never delete the store while
  the daemon is running.
- A cache that cannot stay inside its configured limit is a cache-key or
  capacity problem: check for per-worktree key divergence (different features,
  rustflags, or env between clones) before blaming the cache, since identical
  keys are what let reflinks share one copy across worktrees.

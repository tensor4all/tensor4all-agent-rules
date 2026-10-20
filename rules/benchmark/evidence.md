# Evidence: a number is worth the revision beside it

## Pin from git, never from memory or a tag

A pin is a commit resolved from the repository (`git -C <checkout> rev-parse HEAD`), written into
the pin file by a command, not by hand. A tag is not a pin: a release tag can predate the work a
campaign measures (three campaigns in this workspace were pinned to tags that were ancestors of the
revision they needed, and the build failed or measured the wrong tree). When in doubt, pin `main`'s
head and say which commit it was.

A typed hash is a fabricated hash. Reviewers cannot tell one from a real one, and neither can you
three months later.

## What a run records

A run directory carries the measured checkout (path, commit, dirty state, features), the harness
commit, the toolkit commit, the measured binary's path and SHA-256, the host, the thread
environment, the providers, the guard logs and the raw samples. A page is generated from that
directory; a table typed by hand is not evidence.

## Checked cells and timed cells

A timed cell should be a verified cell. When a verification cannot cover every cell - because
recomputing each case costs more than the timing pass it guards - a suite declares the grid it
verifies (`verify:`), the recorder refuses to widen it, and the manifest records it under
`verification.grid`. The page then says what was checked instead of implying it checked everything.
A budget the verify skipped is never acceptable: a timed number at an unverified budget is an
unchecked number.

## Do not claim a comparison you did not make

If two implementations were compared, say on what values (`sum`, `sum_sq`, `max_abs`, and a
position-weighted sum that a transpose cannot pass). If only an arm's own harness checked it, say
that. "Verified" with no object is the one sentence a benchmark page must never contain.

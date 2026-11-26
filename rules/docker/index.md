# Docker Rules

Apply these rules when adding or changing Docker images, Compose services, or
containerized build, test, and QA runners that share files with the host.

Also read [rootless.md](rootless.md) when configuring or supporting rootless
Docker: setup, UID/GID mapping, daemon selection, storage, and resource limits.

## Host File Ownership

- Files and directories created in host bind mounts must remain owned by the
  invoking host user and readable, editable, and removable without `sudo`.
  This includes build outputs, caches, reports, and retained test fixtures.
- Inspect the image's `USER` and entrypoint, including privilege changes and
  recursive `chown` operations. A service that switches to a fixed non-root UID
  can leave files owned by another host user. Matching usernames alone does
  not align numeric IDs.
- For rootful Linux Docker without user-namespace remapping, run writers with
  the host user's UID and GID, for example with
  `--user "$(id -u):$(id -g)"`, or the equivalent Compose `user` setting.
  Resolve those IDs as the invoking user before any `sudo` or root shell.
- For rootless Docker or user-namespace remapping, account for the actual host
  identity mapping and verify the same ownership contract. Do not assume that
  identical numeric IDs inside and outside the container imply the same user.

## Writable Paths And Mounts

- Make entrypoints executable and configure `HOME`, caches, and output paths
  for the chosen runtime identity. Install system dependencies while building
  the image; choose the writer's runtime UID according to the daemon mode.
- Create host mount directories as the invoking user before starting Docker.
  Prefer explicit `--mount type=bind` declarations that reject missing source
  paths; `-v` can silently create a missing host directory through the daemon,
  leaving it owned by root in a rootful setup.
- Mount source read-only when the workflow permits it and keep writable
  outputs in explicit directories. Container-private state can use a
  disposable volume or temporary filesystem; exports to the host still obey
  the host ownership contract.
- If initialization needs host-root privileges, limit it to explicitly owned
  disposable paths and drop privileges before running builds or tests.
  Document any ownership restoration, cover failure and interruption, and
  report restoration failures. An exit trap alone is insufficient because
  forced termination can skip it.
- Do not repair the design with world-writable permissions, routine
  `sudo rm`, or recursive ownership changes across the whole checkout, home,
  or shared caches. Recovery of existing files must target verified generated
  paths, preserve retained fixtures, and exclude other runs' data.
- Container removal, including `--rm`, does not remove bind-mounted files or
  restore their ownership. Give temporary containers and storage explicit
  cleanup ownership without deleting shared caches or retained fixtures.

## Verification

- Add a focused regression check when changing a runner that writes to host
  paths. Use a disposable bind mount and a non-root invoking host user whose
  UID differs from any image default; exercise the real entrypoint.
- Check generated directories as well as files: inspect numeric ownership and
  verify that the host user can read, modify, and recursively remove the
  disposable outputs without privilege escalation. Deletion depends on the
  containing directory's permissions, not only the file's mode.
- Cover successful execution, command failure, and supported interruption
  paths. Verify the configured cache and artifact paths, and preserve the
  documented fixture-retention behavior.
- Run the bind-mount check on rootful Linux Docker when supporting that
  environment. Docker Desktop or a named-volume-only check can supplement it
  but does not establish the host filesystem behavior. State any unverified
  runtime or identity-mapping configuration.

## References

- [Docker runtime user](https://docs.docker.com/engine/containers/run/#user)
- [Bind mount behavior](https://docs.docker.com/engine/storage/bind-mounts/)
- [User namespace mapping](https://docs.docker.com/engine/security/userns-remap/)
- [Example regression: Synapse QA fixture ownership](https://github.com/shinaoka/koushi-matrix/issues/1570)

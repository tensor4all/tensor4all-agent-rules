# Rootless Docker

Apply when configuring or supporting rootless Docker on Linux. The
[host ownership rules](index.md) still apply; select the runtime identity using
the rootless mapping below.

## Setup On Ubuntu 24.04

These steps assume Docker Engine is installed from Docker's official APT
repository. Run the user-level steps from the intended user's normal login or
SSH session, not a root shell or a session entered with `sudo su`.

1. Install prerequisites with
   `sudo apt install uidmap dbus-user-session docker-ce-rootless-extras`.
   Prefer these packages on Ubuntu 24.04: the packaged RootlessKit path has
   the required AppArmor profile. Do not disable AppArmor or user-namespace
   restrictions globally to work around installation failures.
2. Verify that `/etc/subuid` and `/etc/subgid` each assign at least 65,536
   subordinate IDs to the intended user. Use host-assigned, non-overlapping
   ranges; do not copy another user's ranges or guess an unused range.
3. Inspect existing rootful containers and users before changing the system
   daemon. If retiring it, `sudo systemctl disable --now docker.service
   docker.socket` stops the daemon and disables socket activation; plan for
   disruption of running containers. If it must remain available, the rootless
   setup tool supports `install --force`; keep the contexts explicit.
4. Run `dockerd-rootless-setuptool.sh install` **without sudo**, then
   `systemctl --user enable --now docker` and `docker context use rootless`.
   For startup at boot and operation after logout, enable lingering with
   `sudo loginctl enable-linger "$(id -un)"`.
5. Run `docker info` and `docker run --rm hello-world` without sudo. Require
   `rootless` in the server's Security Options. Check `DOCKER_HOST` and
   `DOCKER_CONTEXT` if an environment override selects another daemon; a
   context name alone does not prove the server is rootless.

## UID/GID Mapping And Shared Files

- Container UID 0 maps to the host user running the rootless daemon. Container
  UID `n >= 1` maps to `subuid_start + n - 1`; GIDs follow `/etc/subgid`.
  Host files owned by that user therefore appear root-owned inside the
  container.
- Do not copy the rootful `--user "$(id -u):$(id -g)"` recipe into a rootless
  runner. Those container IDs map to subordinate host IDs. For writers that
  need the invoking host user's ownership, container UID/GID `0:0` provides
  that mapping without host-root privileges.
- Inspect image `USER` and entrypoint privilege changes. An image that runs
  its service as UID 991, 1000, or another fixed nonzero UID still needs an
  explicit shared-file strategy; rootless mode alone does not align that UID
  with the host user. Configure the writer or keep service-private data in a
  volume and export artifacts with the intended host ownership.
- Verify read, write, and deletion from the host as its ordinary user. Test
  the actual image and bind mounts; a successful `hello-world` run does not
  establish filesystem compatibility.

## Daemon Data, Tools, And Resource Limits

- Rootful and rootless daemons have separate images, containers, and volumes.
  Rootless data defaults to `~/.local/share/docker`; use a supported local
  filesystem, not NFS. Preserve existing rootful data and use an explicit
  export/import or recreation plan when migration is needed.
- Use `systemctl --user` for the rootless service and run Docker commands
  without sudo. Joining the `docker` group does not configure rootless mode.
  CI helpers and child processes must select the intended context or socket;
  do not silently fall back to the system daemon when connection fails.
- Memory, CPU, and PID limits require cgroup v2, the systemd cgroup driver,
  and delegation of the relevant controllers. Check `docker info`, controller
  availability, and effective limits before relying on `--memory`, `--cpus`,
  or `--pids-limit`; do not assume a supplied flag is enforced.
- Validate workload-specific networking and GPU requirements in the selected
  mode. Prefer unprivileged host ports such as 8080 for local tests; document
  any required privileged-port configuration rather than weakening host
  settings globally.

## References

- [Rootless installation](https://docs.docker.com/engine/security/rootless/)
- [Ubuntu prerequisites and troubleshooting](https://docs.docker.com/engine/security/rootless/troubleshoot/)
- [Rootless UID/GID mapping](https://docs.docker.com/engine/security/rootless/uid-gid-mapping/)
- [Contexts, storage, and resource limits](https://docs.docker.com/engine/security/rootless/tips/)

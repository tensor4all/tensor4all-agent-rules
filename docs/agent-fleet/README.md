# Running a Two-Level Agent Fleet

This page describes how the maintainer runs many coding agents at once, and
how to reproduce the setup. It is a guide, not a rule. Adapt it to your own
machines and budget.

![Two-level agent fleet](agent-fleet.svg)

## Architecture

The fleet is a human plus two levels of agents.

- **Level 0: the human.** You decide, approve releases and give instructions
  in plain language. You talk only to the orchestrator. With Claude Code
  [Remote Control](#remote-control-from-a-phone) you can also steer the same
  orchestrator session from a phone. It is the same session, not a second
  one.
- **Level 1: the orchestrator.** One Claude Code session on a laptop. It runs
  rounds on a schedule, reads each agent's screen, dispatches tasks by
  prompt, reviews and merges pull requests, and escalates decisions it should
  not take.
- **Level 2: the machine agents.** Each remote machine runs
  [herdr](https://herdr.dev/docs/), a terminal multiplexer for coding agents,
  with several agents in its panes. Agents are Claude Code sessions (Opus or
  Sonnet) or [pi](https://pi.dev) agents on cheaper models such as DeepSeek.

Two shared services tie the levels together.

- **GitHub.** Agents push branches and open pull requests. When work on an
  issue starts, the issue is assigned to yourself (the person running the
  fleet). Only the orchestrator merges. See
  [GitHub access](#5-github-access) for how to enforce this.
- **A private state repository.** A small git repository holds the recovery
  state, so a new orchestrator session can take over at any time. See
  [Recovery state](#recovery-state).

Each machine has a role. In the maintainer's fleet these are a quiet benchmark
host, an implementation host with a GPU, and a many-core scaling host. Adding
a machine adds a whole team of agents.

## Reproduce the setup

### 1. Network

Install [Tailscale](https://tailscale.com/) on the laptop and on every
machine. The machines then reach each other on a private network and need no
open ports on the public internet.

### 2. SSH key

The orchestrator connects without a human at the keyboard. A passphrase does
not prevent this. Protect the key with a passphrase and load it into
ssh-agent once. After that, every connection uses the loaded key without a
prompt. herdr's own docs give the same advice for its background
connections (see
[Connecting machines](https://herdr.dev/docs/connecting-machines/)).

```sh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_fleet -C fleet
```

On macOS, let the keychain hold the passphrase, so the key is loaded again
after a reboot. Give each machine a Host alias in `~/.ssh/config` on the
laptop:

```sshconfig
Host host-a
    HostName host-a.your-tailnet.ts.net
    User you
    IdentityFile ~/.ssh/id_ed25519_fleet
    IdentitiesOnly yes
    AddKeysToAgent yes
    UseKeychain yes
```

Then load the key once with `ssh-add --apple-use-keychain
~/.ssh/id_ed25519_fleet`. `UseKeychain` exists only on macOS (see Apple's
[TN2449][tn2449]);
on Linux, drop that line and run `ssh-add` once per login.
[`AddKeysToAgent`](https://man.openbsd.org/ssh_config.5) is standard
OpenSSH.

Fallback: a dedicated key without a passphrase (`ssh-keygen -N ""`). It
needs no agent, but anyone who copies the file can log in to every machine.
Use it only if the agent setup does not work for you, and restrict it as
below.

Restrict the key to the tailnet. Tailscale gives every device an address in
[`100.64.0.0/10`](https://tailscale.com/kb/1015/100.x-addresses). In
`~/.ssh/authorized_keys` on each machine, put the
[`from=`](https://man.openbsd.org/sshd.8#from=) option before the key:

```text
from="100.64.0.0/10" ssh-ed25519 AAAA... fleet
```

Also turn off password login in `/etc/ssh/sshd_config`
(`PasswordAuthentication no` and `KbdInteractiveAuthentication no`, see
[sshd_config](https://man.openbsd.org/sshd_config.5)). Optionally, use
[Tailscale SSH](https://tailscale.com/kb/1193/tailscale-ssh) and
[access rules](https://tailscale.com/kb/1018/acls) to decide which devices
may connect to which machines.

Check that `ssh host-a true` works without any prompt.

### 3. herdr

Install herdr on the laptop
([install docs](https://herdr.dev/docs/install/)):

```sh
curl -fsSL https://herdr.dev/install.sh | sh
```

Then register each machine once, in an interactive terminal on the laptop:

```sh
herdr machine add host-a
herdr machine list
```

`machine add` takes an SSH target, normally the Host alias from
`~/.ssh/config`. It must run in an interactive terminal, because it may ask
before it installs or replaces anything. If herdr is missing on the machine,
it offers to install it there. It then starts the remote background server
and saves a profile. The profile's label defaults to the SSH host without
any `user@` prefix, here `host-a`. That label (or the profile id) is what
`--machine` accepts. An arbitrary hostname does not work. Add
`--label <name>` to choose another label. See
[Connecting machines](https://herdr.dev/docs/connecting-machines/).

The remote server ignores SIGHUP, so it keeps running after your SSH session
ends ([Persistence](https://herdr.dev/docs/persistence-remote/)). Nothing
restarts it after a reboot. A reboot also ends the agents in its panes;
herdr restores the saved layout, not the running processes
([Session state](https://herdr.dev/docs/session-state/)). If a machine
reboots often, run the server under systemd. The CLI reference provides
`herdr server` for this: it "runs the headless server explicitly" for
supervised setups. A minimal user unit, `~/.config/systemd/user/herdr.service`
on the machine:

```ini
[Unit]
Description=herdr server

[Service]
ExecStart=%h/.local/bin/herdr server
Restart=on-failure

[Install]
WantedBy=default.target
```

Adjust the path to the output of `command -v herdr`. Enable it with
`systemctl --user enable --now herdr.service`, and run
[`loginctl enable-linger`][loginctl]
once so the user manager starts at boot. Panes inherit the service
environment, so add `Environment=` lines if tools are missing from `PATH`.
Enable the unit before `machine add`, or stop the server that `machine add`
started (`herdr server stop` on the machine) before you enable it. herdr's
docs do not ship such a unit. Treat it as a starting point.

Adding and removing machines is the human's job. The orchestrator only uses
machines that are already registered.

### 4. Orchestrator commands

The orchestrator drives every machine with `herdr --machine <label> ...`.
These are the forms it needs, from the
[CLI reference](https://herdr.dev/docs/cli-reference/). Put
`herdr --machine <label>` in front of each one, except `machine status`. A
target is an agent name or a pane id such as `w1:p2`.

| Command | Use |
|---|---|
| `herdr machine status [<label>]` | Check SSH and the remote server. |
| `agent list` | List the agents and their state. |
| `agent get <target>` | Show one agent's details. |
| `agent read <target> [--source recent-unwrapped] [--lines N]` | Read the agent's screen. |
| `agent prompt <target> <text> [--wait] [--timeout MS]` | Send a task or an answer. |
| `agent send-keys <target> <key> [key ...]` | Send keys such as `esc`, `enter`, `shift+tab`. |
| `agent wait <target> [--until STATUS] [--timeout MS]` | Wait for `idle`, `done` or `blocked`. |
| `agent rename <target> <name>` | Name an agent. |
| `pane list` | List panes and their ids. |
| `pane split <pane_id> --direction right\|down [--cwd PATH] [--no-focus]` | Make a pane for another agent. |
| `agent start <name> --kind KIND --pane <pane_id> [-- <agent-args...>]` | Start an agent in an empty shell pane. |

To start a new agent, first make a pane, then start the agent in it.
`agent start` never creates panes. Use an absolute path for `--cwd`:

```sh
herdr --machine host-a pane list
herdr --machine host-a pane split w1:p1 --direction right \
  --cwd /home/you/src/tenferro-rs --no-focus
# The JSON reply holds the new pane id in .result.pane.pane_id, say w1:p2.
herdr --machine host-a agent start host-a-tenferro-impl --kind claude \
  --pane w1:p2 -- --allow-dangerously-skip-permissions --model opus
```

For a pi agent, use `--kind pi` and pass pi's flags after `--`, for example
`-- --model <provider>/<model-id>`.

A failed connection or a timeout does not prove that a prompt was not
delivered. herdr does not retry it either. Read the agent's screen before you
send it again.

### 5. GitHub access

The agents need GitHub access on each machine. Run
[`gh auth login`](https://cli.github.com/manual/gh_auth_login) there, or set
`GH_TOKEN` to a
[fine-grained token][gh-pat]
(gh's docs prefer `GH_TOKEN` for fine-grained tokens). Then run
[`gh auth setup-git`](https://cli.github.com/manual/gh_auth_setup-git) so
`git push` uses the same credentials. herdr also documents SSH agent
forwarding (`ForwardAgent yes`) for git inside remote panes. It lends your
laptop's keys to every agent on that machine, so prefer a token.

Enforce "only the orchestrator merges" on GitHub, not only in prompts.
Merging a pull request needs the same "Contents: write" permission as
pushing a branch
([permissions table][gh-pat-perms]),
so a token alone cannot allow pushes and forbid merges. Instead:

- Give the agents a separate GitHub account with the Write role, and give
  their tokens access to the needed repositories only. Never give them the
  Admin role.
- On each default branch, add a
  [ruleset][gh-ruleset]
  with "Restrict updates" and "Require a pull request before merging"
  ([rules][gh-rules]).
  Put only the role your own account holds, such as Maintain or Admin, on
  the bypass list, set to "For pull requests only".
- The orchestrator merges with your own account on the laptop.
- Never run [`cargo login`][cargo-login]
  on an agent machine. Keep the crates.io token on the laptop only.

### 6. Agent names

Name every agent `<machine>-<repo>-<role>[-<model>]`, for example
`host-b-tenferro-impl` or `host-c-t4a-review-dsv4f`. Use lowercase letters,
digits, `-` and `_`, start with a letter, and use at most 32 characters.
Claude agents may omit the model suffix. Names disappear when an agent
exits, so check them on every round. Refer to agents by name, never by pane
id.

### 7. Claude Code agents

Start Claude agents with `agent start --kind claude`, as in
[section 4](#4-orchestrator-commands), and pass Claude's flags after `--`:

```sh
-- --allow-dangerously-skip-permissions --model opus    # or sonnet
```

`--kind` runs herdr's own `claude` executable, so your shell aliases do not
apply. Pass the flags in full.
[`--allow-dangerously-skip-permissions`][cc-cli]
adds bypass mode to the Shift+Tab cycle without starting in it. When the
agent starts, accept any startup dialog, then send `shift+tab` until the
status line shows that bypass permissions is on. Read the
[permission modes](https://code.claude.com/docs/en/permission-modes) page
first, and use bypass mode only on machines you control.

### 8. pi agents

[pi](https://github.com/earendil-works/pi) runs agents on cheaper models.
Install it on each machine
([quickstart][pi-quickstart]):

```sh
curl -fsSL https://pi.dev/install.sh | sh
```

pi needs Node.js 22.19 or newer. The installer can install Node.js if it is
missing. Update later with `pi update`.

Log in once: start `pi` and run `/login`, or export the provider's key, for
example `DEEPSEEK_API_KEY`
([providers][pi-providers]).
`/login` saves credentials in `~/.pi/agent/auth.json`. Keep that file
private.

pi loads skills from `~/.pi/agent/skills/`, from `.pi/skills/` in the
project, and from the shared `~/.agents/skills/` and `.agents/skills/`
([skills][pi-skills]).

Two things differ from Claude Code.

- A prompt sent while the agent is in a tool call waits until that tool call
  ends. For an urgent prompt, send `esc` first (it aborts the run), then the
  prompt.
- The screen shows little history. Read the session log instead, in
  `~/.pi/agent/sessions/*/*.jsonl` on that machine
  ([sessions][pi-sessions]).

### 9. Model policy

Use Sonnet or a cheap model for simple work: docs, mechanical edits, running
benchmarks, collecting logs. Use Opus for complex coding, reviews and
judgment calls. The orchestrator itself runs on the strongest model you
have.

### 10. Scheduled rounds

The orchestrator's rounds are Claude Code
[scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks). Start
one with `/loop`, for example `/loop 30m <round prompt>`, or ask Claude to
schedule it; Claude then uses the `CronCreate` tool. Know the limits:

- The tasks belong to that one session. They fire only while that `claude`
  process runs and is idle.
- Recurring tasks expire 7 days after creation.
- The laptop must stay awake. On macOS, `caffeinate -i` keeps it from
  sleeping while it runs.
- `claude --resume` restores unexpired tasks, but not a self-paced `/loop`.

Keep every round prompt and its schedule in `monitoring.md` (see
[Recovery state](#recovery-state)). After a new session, after a resume, or
before the 7 days are up, recreate the tasks from that file.

### 11. Remote Control from a phone

To steer the orchestrator from a phone, use Claude Code
[Remote Control](https://code.claude.com/docs/en/remote-control).

1. In the orchestrator session, run `/remote-control`. Or start the session
   with `claude --remote-control`.
2. On the phone, open the Claude app, tap Code, and pick the session. Or
   open `claude.ai/code` in a browser.

Remote Control needs a Pro, Max, Team or Enterprise plan and a claude.ai
login. API keys do not work. On Team and Enterprise, an Owner must turn it
on first. The session still runs on the laptop, so the laptop and the
`claude` process must keep running.

### 12. Cost and limits

An Opus orchestrator on a 10-minute round, plus several Claude agents, uses
up subscription limits quickly. pi agents bill per token through the
provider's API key. Start with 2 or 3 agents and a 30-minute round. Shorten
the round and add agents once you know your usage.

## Orchestrator operating rules

These are the rules the maintainer's orchestrator follows. Write your own
version into the orchestrator's instructions.

- **Rounds.** In each round, read every agent, unblock or redirect, and merge
  what is ready. The maintainer uses a 10-minute round; see
  [Cost and limits](#12-cost-and-limits) before you copy that. Twice an
  hour, check overall progress against the TODO queue.
- **Issue patrol.** Once an hour, scan the tracked repositories for new
  issues and comments.
- **Daily checks.** Update Claude Code (`claude update`), pi (`pi update`)
  and herdr (`herdr update`) on every machine. Read the issue trackers of
  upstream projects you depend on.
- **Approval dialogs.** Never answer an agent's approval dialog on its
  behalf. Report it to the human instead. The only exceptions are dialogs
  agreed in advance, such as folder trust, a model switch, or accepting
  bypass permissions at startup.
- **Releases.** Publish to crates.io only after you approve that release
  explicitly in the conversation.
- **GPU work.** Validate GPU changes on a local GPU instead of running paid
  GPU CI.
- **Issue assignment.** When an agent starts on an issue, assign the issue to
  yourself (the person running the fleet). This avoids duplicate work with
  outside contributors.
- **Merging.** Use a merge commit when the branch contains other people's
  commits, so their authorship is kept.
- **Benchmarks.** Measure on a quiet host with CPU boost off. See
  [`rules/benchmark`](../../rules/benchmark/index.md).
- **Approvals.** Record every approval word for word, with its date, in the
  handoff file.

## Recovery state

A long orchestrator session dies or fills its context sooner or later. The
recovery state lets a fresh session take over in a few minutes. Keep it in
a private git repository that every machine can push to.

```text
state/<project>/
  handoff.md            written only by the orchestrator
  monitoring.md         round prompts and their schedules
  agents/<name>.md      each agent writes only its own file
  worklog/<date>.md     append-only, anyone
```

- `handoff.md` is the current picture of the fleet. The orchestrator
  replaces its content whenever something changes, instead of adding to the
  end. See the skeleton below.
- `agents/<name>.md` belongs to one agent. It says what the agent works on,
  where (branch and worktree), what it did last, what it does next, and what
  blocks it. Only that agent writes the file, so agents on different
  machines never edit the same file.
- `worklog/<date>.md` is a dated diary. Anyone may add a line about a merge,
  a release, an incident, or a decision and who made it. Nobody edits old
  lines.
- `monitoring.md` keeps each scheduled prompt word for word, with its
  schedule, so a new session can set up the same rounds.

A skeleton `handoff.md`:

```markdown
# Handoff

## Machines
- host-a: benchmark host, quiet, boost off

## Agents
- host-a-tenferro-bench: running the einsum benchmark for PR #123

## Open PRs
- tenferro-rs#123: CI green, waiting for benchmark numbers

## Tracked issues
- tenferro-rs#100: next after #123 merges

## Waiting for the human
- Whether to drop the old API in the next minor release

## Approvals
- 2026-01-15: "Release 0.3.0 to crates.io." (approved in chat)

## Standing rules
- Never run paid GPU CI.

## TODO queue
1. Merge #123 after the benchmark.
```

An example `monitoring.md` entry:

```markdown
## Round (every 30 minutes)
/loop 30m Read state/myproject/handoff.md. For each agent listed there, run
`herdr --machine <label> agent read <name> --lines 80`.
Unblock or redirect agents, merge PRs that are ready, then update handoff.md.
```

A short first prompt for a new orchestrator session:

```text
You are the orchestrator for this fleet. Read state/myproject/handoff.md,
every file in state/myproject/agents/, and the last two worklog files. Check
each agent with herdr agent get and agent read, and each open PR with gh,
before you act. Then recreate the rounds in state/myproject/monitoring.md
and report what you found.
```

To take over after a crash or a full context:

1. Read the handoff file first. Then read each agent's file, then the two
   most recent worklogs.
2. Before you act, check the real state: each agent with `agent get` and
   `agent read`, and each open PR on GitHub. The files may miss the latest
   change.
3. Set up the scheduled rounds again from `monitoring.md`.

## How the maintainer's rule set is built

The maintainer's own rules are private, but their structure is easy to copy.
There are four layers.

1. **Global instructions** in `~/.claude/CLAUDE.md`. Personal preferences
   that hold in every project: the response language, preferred tools (for
   example CodeGraph before grep), and limits on remote control (which herdr
   commands the session may run, and that it never answers another agent's
   approval dialog).
2. **Per-repository `AGENTS.md`.** Each tensor4all repository keeps a short
   `AGENTS.md` that points to this repository for shared rules and adds only
   repository-specific ones.
3. **Claude Code auto-memory.** A memory directory with one fact per file
   and an index file that lists them all, one line each. Each fact has a
   type: user, feedback, project or reference. Feedback and project entries
   carry a "Why" line and a "How to apply" line, so the rule can be judged
   later rather than followed blindly.
4. **A private advisor repository.** It holds:
   - **Skills.** Claude Code loads personal skills from
     `~/.claude/skills/<name>/SKILL.md` and project skills from
     `.claude/skills/<name>/SKILL.md`
     ([skills](https://code.claude.com/docs/en/skills)). An `autopilot`
     skill runs one issue or phase alone. It plans the phase, drives cheap
     workers through pi with a supervisor script and a worker launcher, gets
     an independent review from a different model, opens one PR and fixes CI
     until it is green. A worker launcher can run pi without its UI, for
     example `pi --print --model <provider>/<model-id> "<task>"`, or
     `pi --mode json` for an event stream
     ([pi CLI][pi-cli]).
     A `build-profile` skill builds the decision profile below. These skills
     are not published, so this page shows their shape, not their code.
   - **A decision profile.** `profile/rules.md` holds rules, and a
     precedents file holds short past decisions with a pointer to their
     source. The `build-profile` skill mines past transcripts and worklogs
     for decisions that recur, and turns them into rules. Rules that came
     from the maintainer directly are never weakened by a rebuild.
   - **Proxy logs.** Every decision the autopilot takes on the maintainer's
     behalf is logged. The maintainer reviews and corrects them, and the
     corrections feed the next profile update.
   - **The `state/` directory** described in
     [Recovery state](#recovery-state).

Start small. A global `CLAUDE.md`, an `AGENTS.md` per repository and the
state directory are enough for a fleet of a few agents. Add the profile when
you notice that you answer the same questions again and again.

## Security notes

- Bypass permissions are acceptable only on machines you control, inside
  the tailnet. Restrict the SSH key with `from=` as shown above, and do not
  reuse it elsewhere.
- Agent machines hold only a limited GitHub token and no crates.io token.
  See [GitHub access](#5-github-access).
- Keep secrets out of every file an agent can read: the state repository,
  worklogs, memory and prompts. Agents copy what they see into logs and PRs.
- On a shared machine, other users' jobs come first. Check the load before
  starting builds or benchmarks, limit the cores your agents use, and never
  kill processes you do not own.

[tn2449]: https://developer.apple.com/library/archive/technotes/tn2449/_index.html
[loginctl]: https://www.freedesktop.org/software/systemd/man/latest/loginctl.html
[gh-pat]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
[gh-pat-perms]: https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens
[gh-ruleset]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository
[gh-rules]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets
[cargo-login]: https://doc.rust-lang.org/cargo/commands/cargo-login.html
[cc-cli]: https://code.claude.com/docs/en/cli-reference
[pi-quickstart]: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/quickstart.md
[pi-providers]: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md
[pi-skills]: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/skills.md
[pi-sessions]: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sessions.md
[pi-cli]: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/cli.md

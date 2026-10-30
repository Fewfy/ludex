# ludex

A resident daemon that hunts down and cleans up zombie agent processes.

The name comes from *Iudex Gundyr*, the first boss of Dark Souls III. *Iudex* is Latin for
"judge" — ludex runs the same trial on the agent processes your machine forgot to bury.

## The problem

Coding agents (Claude Code, Codex, and friends) leak processes. A crashed terminal, an abrupt
disconnect, a session that timed out — any of these can leave an agent running with no parent
to reap it. The leftovers sit there holding memory, file descriptors, ports, and lock files,
sometimes still writing logs. Over time they accumulate, and the next session starts in a
crowded, inconsistent environment.

ludex watches for these orphans continuously and puts them down.

> "Zombie" here means an **orphaned or abandoned agent process**, not a process in the POSIX
> `<defunct>` state.

## How it judges

On every sweep, ludex enumerates processes that match a known agent executable or command-line
signature, then marks a process as a candidate when any of the following holds:

- its parent has exited and it has been reparented to init (`PPID = 1`);
- its controlling terminal or session no longer exists;
- it has been idle for longer than the configured threshold — no CPU activity and no children;
- its heartbeat file is stale (for agents that publish one).

Candidates are reconsidered for one grace period: if the process exits or shows activity within
it, it is spared. Otherwise it receives `SIGTERM`, and `SIGKILL` if it still refuses to die after
the kill timeout.

Active sessions, whitelisted processes, and ludex itself are never candidates. Exemptions
always take precedence over the detection rules.

## Install and run

ludex runs as a background service:

```bash
ludex install     # register as a launchd / systemd user service
ludex start
ludex status      # daemon state, last sweep, processes reaped
```

For debugging, run it in the foreground:

```bash
ludex run --foreground
ludex run --dry-run       # log what would be killed, touch nothing
```

## Control

```bash
ludex scan                # trigger an immediate sweep and print the report
ludex list                # show candidates currently being tracked
ludex exempt <pid>        # mark a process as untouchable
```

## Configuration

`~/.config/ludex/config.toml`:

```toml
[daemon]
interval = "60s"          # time between sweeps
kill_timeout = "10s"      # SIGTERM → SIGKILL escalation
grace_period = "5s"       # observation window before signaling

[detection]
idle_threshold = "10m"
agents = ["claude", "codex", "opencode"]

[exempt]
pids = []
commands = []             # regexes matched against the full command line
```

## Safety

- Signals only processes that match an agent signature — never a PID range.
- Exemptions and active sessions override every detection rule.
- `--dry-run` is available end to end; the default grace period means a process has to be idle
  and parentless across more than one sweep before it is touched.
- Every reaping decision is logged with PID, command line, matched rule, and signal sent.

## Name

*Iudex Gundyr* — the Ashen Judge. Spelled `ludex` here, but the job is the same: stand at the
gate and let nothing dead walk past.

# ludex — Agent Instructions

ludex is a resident daemon that finds and reaps orphaned agent processes. The name comes from
*Iudex Gundyr*, the first boss of Dark Souls III; *iudex* is Latin for "judge". `README.md` is the
user-facing description and the source of truth for CLI and config surface. This file is the entry
point for every coding agent working in the repository.

`CLAUDE.md` is a compatibility shim and contains only `@AGENTS.md`. Do not maintain it separately.

**Status:** the repository is scaffold-only — `README.md` plus these instruction files. The layout
described below is the target structure; create directories as the corresponding code lands.

## Workflow

- Read this file and `README.md` before changing anything.
- Implementation work starts with a failing test. Write the test, watch it fail for the expected
  reason, then implement. No test, no feature.
- For non-trivial changes, state the plan before editing files.
- Make the smallest change that fixes the root cause. Do not refactor unrelated code.
- Run `make fmt`, `make lint`, and `make test` before claiming work is done. If a target does not
  exist yet, create it rather than running the underlying command ad hoc.
- Preserve the user's existing changes. Never overwrite, reset, or discard them.

## Source of truth

| Content | Source |
|---|---|
| User-facing behavior, CLI, config keys | `README.md` |
| Repository conventions and safety rules | this file |
| Actual behavior | code and tests |
| Local commands | `Makefile` |
| CI gates | `.github/workflows/` |

When code and `README.md` disagree, treat it as a bug in one of them and fix it explicitly rather
than silently treating either as the new contract.

## Inviolable rules

These are safety-critical. Violating any of them is a bug regardless of intent.

- Signal only processes that match a configured agent signature. Never signal by PID range, and
  never signal "everything that looks idle".
- Exemptions — active sessions, the whitelist, and ludex itself — always override detection.
- Every reap is logged with pid, full command line, matched rule, and the signal sent.
- `--dry-run` must remain a complete no-op end to end, including in the daemon loop.
- Default to `SIGTERM` with a grace period; escalate to `SIGKILL` only after the timeout expires.
- When in doubt, escalate too slowly rather than kill too fast. A missed cleanup is recoverable;
  a wrong kill is not.
- Run as a user-level service (launchd LaunchAgent, `systemd --user`). Do not require root.
- Never use `--no-verify` or `--no-gpg-sign`. Never force-push a shared branch.

## Architecture boundaries

The correctness risk in this project is killing the wrong process, not performance. Structure the
code so that risk is testable and small.

- Detection is a pure function: a process snapshot in, candidates out. No I/O, no signals, no
  clock reads inside it — pass whatever it needs in. Test it exhaustively against fakes.
- Process enumeration and signaling sit behind interfaces so tests never touch real processes. The
  dry-run signaler is a second implementation of the same interface, not a flag threaded through
  the real one.
- Prefer `gopsutil` for the portable path (pid, ppid, cmdline, create time, CPU times). Isolate
  platform-specific checks — controlling terminal and session liveness — in files behind the shared
  interface rather than branching throughout the code.
- Config is loaded and validated once at startup. The sweep loop receives plain structs.

## Go conventions

- Module path: `github.com/Fewfy/ludex`.
- Layout: `cmd/ludex/` holds `main`; everything else lives under `internal/`.
- Code comments and documentation are written in English; identifiers stay English regardless.
- Do not add a dependency without stating why the standard library or an existing dependency cannot
  do the job.
- Wrap errors with context (`fmt.Errorf("scan processes: %w", err)`). Do not swallow errors or
  discard them into logs without returning.
- Use `log/slog` for structured records; process decisions are structured data, not prose.

## Configuration

- TOML at `$XDG_CONFIG_HOME/ludex/config.toml`, falling back to `~/.config/ludex/config.toml`.
  Keys are snake_case.
- Interval, grace period, kill timeout, idle threshold, agent signatures, and exemptions are all
  configurable. Do not hard-code them, and do not read the process environment to fill or validate
  config; resolve everything through the config file and explicit defaults.

## Commands

`make` is the single local entry point. Target set:

```text
make fmt      # gofmt / gofumpt
make lint     # go vet + staticcheck
make test     # unit tests; must not require real agent processes
make build    # -> bin/ludex
make run      # foreground run for debugging
make clean
```

CI calls the same targets; do not write a second set of commands in workflow YAML.

## Branches and commits

- Commit messages follow Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `chore:`) and
  are written in English.
- `main` is the deliverable branch. Changes to process enumeration, detection, or signaling go on a
  `feat/*` or `fix/*` branch and through a pull request — that is the code that can kill something
  it should not. Docs and config-value tweaks may land on `main` directly.
- Use `gh` for pull request operations.
- Push only when the user has explicitly asked in the current session.

## Repository map

| Path | Status | Purpose |
|---|---|---|
| `cmd/ludex/` | planned | binary entry point, flags, service subcommands |
| `internal/agent/` | planned | known agent signatures and matching |
| `internal/scan/` | planned | process snapshot via gopsutil; per-platform session checks |
| `internal/judge/` | planned | pure detection: snapshot → candidates |
| `internal/reap/` | planned | signaler interface, escalation, dry-run implementation |
| `internal/config/` | planned | TOML loading and defaults |
| `internal/service/` | planned | launchd plist and systemd unit templates |
| `Makefile` | planned | local entry point |
| `.github/workflows/` | planned | CI: fmt, lint, test |
| `README.md` | present | user-facing description |
| `AGENTS.md`, `CLAUDE.md` | present | agent instructions and shim |

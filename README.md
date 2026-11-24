# hermes

Run a long command in the background and get GNOME notifications as it progresses, so you don't babysit a terminal (or, if you're driving an AI coding agent, so the agent doesn't burn tokens polling for progress). Named after the Greek messenger: it runs far and comes back with news.

Pure Bash. No runtime, no daemon, no dependencies beyond standard Linux tools.

## What it does

- Runs your command **detached** (`setsid`) so it survives your terminal/session closing.
- Reads progress that **your command reports** and fires a desktop notification every 15% with an ETA.
- Sends a final notification when the job finishes: `✅ 100% • done` or `❌ failed exit=N`. (100% is signalled by this final notification, not by a separate progress popup.)
- Records each run under `~/.hermes/<timestamp>-<slug>/` so you can check results later.

It does **not** try to guess progress. Your command owns the percentage; hermes just reads it and notifies. If a command reports no progress, hermes falls back to a plain "still running, N min elapsed" ping every 10 minutes (no fake ETA).

## Requirements

- Linux with a desktop that shows `notify-send` notifications (GNOME, KDE, etc.).
- `notify-send` — Debian/Ubuntu: `sudo apt install libnotify-bin`. Fedora: `sudo dnf install libnotify`.
- `setsid` — part of `util-linux`, present on virtually all distros.
- Bash + coreutils.

## Install

Copy the `hermes` script somewhere on your `PATH` and make it executable:

```bash
install -Dm755 hermes ~/.local/bin/hermes
```

Kiro users: also install the steering file so the agent knows when to use hermes:

```bash
install -Dm644 kiro/steering/hermes.md ~/.kiro/steering/hermes.md
```

Make sure `~/.local/bin` is on your `PATH` (most distros add it automatically). Check:

```bash
which hermes
```

## Validate the install

```bash
hermes selftest
```

This runs a real ~12-second job that reports progress. Watch your desktop: you should see progress notifications with an ETA, then a final "✅ done" popup. The command also prints `PASS` / `FAIL` for the recorded run state. If `notify-send` or `setsid` are missing, it tells you which.

## Usage

```bash
hermes "<short description>" <command> [args...]
```

Example:

```bash
hermes "nightly backfill" ./backfill.sh --from 2024-01-01
```

hermes prints the PID and a log path and returns immediately — your shell is free. You'll be notified as it progresses and when it finishes.

### Check on jobs

```bash
hermes status
```

Lists recent runs with their state (`RUNNING` / `done exit=N` / `dead`) and the log path. The full output of each command is in `~/.hermes/<run>/log`.

## Make your scripts report progress

For a real ETA, your script must print its own progress. hermes reads lines of exactly this form from stdout:

```
PROGRESS <0-100> [optional message]
```

Emit it at meaningful checkpoints (per batch / file / step), not on every tiny iteration.

Bash example:

```bash
total=$#; i=0
for item in "$@"; do
  process "$item"
  i=$((i+1))
  echo "PROGRESS $(( i * 100 / total )) processed $i/$total"
done
echo "PROGRESS 100 done"
```

Python example:

```python
for i, item in enumerate(items, 1):
    process(item)
    print(f"PROGRESS {i * 100 // len(items)} processed {i}/{len(items)}", flush=True)
```

(Use `flush=True`, or run Python with `-u`, so progress isn't stuck in a buffer.)

## Where state lives

Each run creates `~/.hermes/<timestamp>-<slug>/`:

- `log` — the command's full stdout + stderr.
- `meta` — `desc`, `cmd`, `started`, `pid`, `finished`, `exit`, and optional `goal`.

To tag a run with what you were trying to accomplish, set `HERMES_GOAL`:

```bash
HERMES_GOAL="rebuild search index for release" hermes "reindex" ./reindex.sh
```

Old run directories are just folders — delete them whenever you like: `rm -rf ~/.hermes/<run>`.

## Configuration (environment variables)

| Variable | Default | Meaning |
|---|---|---|
| `HERMES_HOME` | `~/.hermes` | Where run state is stored |
| `HERMES_STEP` | `15` | Notify every N percent |
| `HERMES_HEARTBEAT_MIN` | `10` | Fallback ping interval (minutes) when no progress is reported |
| `HERMES_GOAL` | *(unset)* | Free-text goal saved into the run's `meta` |

## When to use it (and when not)

Use it for commands that **run and exit** and take long enough that waiting is wasteful: big builds, data backfills, large downloads, batch jobs, long test suites.

Don't use it for interactive commands (they can't prompt you in the background) or long-lived servers (they never "finish").

## Limitations

- Jobs survive your session closing but **not a machine reboot** — there is no auto-restart. For durable, restartable jobs, use a real supervisor like a systemd user unit.
- The monitor polls the log every 2 seconds, so a job that races through milestones in a few seconds may skip some 15% notifications. For real multi-minute jobs this never matters.
- ETA assumes progress is roughly linear in time; a job that stalls then sprints will show a rough estimate.

## How it works (internals)

1. The launcher creates the run directory, then `setsid`s a small supervisor that runs your command, pipes its output to `log`, and writes `finished` + `exit` to `meta` when done.
2. A second `setsid`'d process is the monitor: it tails `log`, parses the latest `PROGRESS N`, and calls `notify-send` when a new 15% bucket is crossed, computing ETA from elapsed time and reported percent. When the command's process exits, it reads the exit code and sends the final notification.

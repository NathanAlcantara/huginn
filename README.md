# huginn

Run a long command in the background and get GNOME notifications as it progresses, so you don't babysit a terminal (or, if you're driving an AI coding agent, so the agent doesn't burn tokens polling for progress). Named after Huginn, one of Odin's ravens, which flies over the world every day and comes back to tell what it saw.

Pure Bash. No runtime, no daemon, no dependencies beyond standard Linux tools.

## What it does

- Runs your command **detached** (`setsid`) so it survives your terminal/session closing.
- Reads progress that **your command reports** and fires a desktop notification every 15% with an ETA.
- Sends a final notification when the job finishes: `✅ 100% • done` or `❌ failed exit=N`. (100% is signalled by this final notification, not by a separate progress popup.)
- Records each run under `~/.local/state/huginn/<timestamp>-<slug>/` so you can check results later.

It does **not** try to guess progress. Your command owns the percentage; huginn just reads it and notifies. If a command reports no progress, huginn falls back to a plain "still running, N min elapsed" ping every 10 minutes (no fake ETA).

## Requirements

- Linux with a desktop that shows `notify-send` notifications (GNOME, KDE, etc.).
- `notify-send` — Debian/Ubuntu: `sudo apt install libnotify-bin`. Fedora: `sudo dnf install libnotify`.
- `setsid` — part of `util-linux`, present on virtually all distros.
- Bash + coreutils.

## Install

Install by symlinking from a clone of this repo, so a `git pull` updates the installed copy:

```bash
git clone https://github.com/NathanAlcantara/huginn.git ~/Documents/huginn   # or wherever you keep it
cd ~/Documents/huginn
mkdir -p ~/.local/bin
ln -sf "$PWD/huginn" ~/.local/bin/huginn
ln -sf "$PWD/README.md" ~/.local/bin/huginn.README.md   # optional
```

Kiro users: also link the steering file, so the agent knows when and how to use huginn:

```bash
mkdir -p ~/.kiro/steering
ln -sf "$PWD/kiro/steering/huginn.md" ~/.kiro/steering/huginn.md
```

Claude Code users: link the skill, so Claude uses huginn for long jobs:

```bash
mkdir -p ~/.claude/skills
ln -sfn "$PWD/claude/skills/huginn" ~/.claude/skills/huginn
```

Edit the files in the repo, not through the links. If you move or delete the repo, the `huginn` command breaks: run the `ln -sf` lines again from the new location.

Make sure `~/.local/bin` is on your `PATH` (most distros add it automatically). Check:

```bash
which huginn
```

## Validate the install

```bash
huginn selftest
```

This runs a real ~12-second job that reports progress. Watch your desktop: you should see progress notifications with an ETA, then a final "✅ done" popup. The command also prints `PASS` / `FAIL` for the recorded run state. If `notify-send` or `setsid` are missing, it tells you which.

## Usage

```bash
huginn "<short description>" <command> [args...]
```

Example:

```bash
huginn "nightly backfill" ./backfill.sh --from 2024-01-01
```

huginn prints the PID and a log path and returns immediately — your shell is free. You'll be notified as it progresses and when it finishes.

### Check on jobs

```bash
huginn status
```

Lists recent runs with their state (`RUNNING` / `done exit=N` / `dead`), the run id and the log path. The full output of each command is in `~/.local/state/huginn/<run>/log`.

### Stop a job

```bash
huginn stop <run id>
```

Sends `SIGTERM` to the command and everything it started (its whole process group) and records `exit=stopped`. Pass the run id from `huginn status` or the run directory path. Don't `kill` the pid from `meta` yourself: it is the supervisor, and killing it alone leaves the command running.

### Clean up old runs

```bash
huginn prune      # delete finished runs older than 30 days
huginn prune 7    # ... older than 7 days
```

Running and unfinished runs are never deleted.

## Make your scripts report progress

For a real ETA, your script must print its own progress. huginn reads lines of exactly this form from stdout:

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

Each run creates `~/.local/state/huginn/<timestamp>-<slug>/` (`$XDG_STATE_HOME/huginn` if that is set):

- `log` — the command's full stdout + stderr.
- `meta` — `desc`, `cmd`, `started`, `pid`, `finished`, `exit`, and optional `goal`.

To tag a run with what you were trying to accomplish, set `HUGINN_GOAL`:

```bash
HUGINN_GOAL="rebuild search index for release" huginn "reindex" ./reindex.sh
```

Old runs: `huginn prune`, or delete a run folder by hand.

## Configuration (environment variables)

| Variable | Default | Meaning |
|---|---|---|
| `HUGINN_HOME` | `~/.local/state/huginn` | Where run state is stored |
| `HUGINN_STEP` | `15` | Notify every N percent |
| `HUGINN_HEARTBEAT_MIN` | `10` | Fallback ping interval (minutes) when no progress is reported |
| `HUGINN_GOAL` | *(unset)* | Free-text goal saved into the run's `meta` |

## When to use it (and when not)

Use it for commands that **run and exit** and take long enough that waiting is wasteful: big builds, data backfills, large downloads, batch jobs, long test suites.

Don't use it for interactive commands (they can't prompt you in the background) or long-lived servers (they never "finish").

## Limitations

- Jobs survive your session closing but **not a machine reboot** — there is no auto-restart. For durable, restartable jobs, use a real supervisor like a systemd user unit.
- The monitor polls the log every 2 seconds, so a job that races through milestones in a few seconds may skip some 15% notifications. For real multi-minute jobs this never matters.
- ETA assumes progress is roughly linear in time; a job that stalls then sprints will show a rough estimate.
- `huginn stop` sends only `SIGTERM`. A command that ignores it keeps running; kill it by hand.

## How it works (internals)

1. The launcher creates the run directory, then `setsid`s a small supervisor that runs your command, pipes its output to `log`, and writes `finished` + `exit` to `meta` when done.
2. A second `setsid`'d process is the monitor: it tails `log`, parses the latest `PROGRESS N`, and calls `notify-send` when a new 15% bucket is crossed, computing ETA from elapsed time and reported percent. When the command's process exits, it reads the exit code and sends the final notification.

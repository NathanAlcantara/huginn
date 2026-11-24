---
name: huginn
description: Run long commands (likely over ~10 minutes, or that should outlive this session) detached through the `huginn` wrapper, which sends GNOME notifications with progress and ETA. Use when about to launch a long build, backfill, batch job, data migration, large download or big test suite; when writing a script that will run for a long time; or when the user asks about, resumes or wants to cancel a huginn job.
---

# Huginn: long-running background jobs

`huginn` (at `~/.local/bin/huginn`, a symlink into the huginn git repo) runs a command detached with `setsid`, so it survives this Claude session closing. It sends a GNOME notification every 15% with an ETA and a final ✅/❌ when the command exits. The user can leave and gets pinged; nobody spends tokens polling.

## Huginn or Bash `run_in_background`?

- Use **huginn** when the job will likely take more than ~10 minutes, the user may close the session or walk away, or the job must keep running after this session ends.
- Use the Bash tool's `run_in_background` for shorter jobs whose result you need later in this same turn or session.
- Only for commands that run and exit. Never for interactive commands or servers.

## Launch

```bash
HUGINN_GOAL="<what the larger task is trying to achieve>" huginn "<short description>" <command> [args...]
```

It prints the pid, the log path and returns at once. Then tell the user the job is running in the background and they will be notified. Do not wait on it, `sleep`, poll the log, or schedule check-ins. Continue with the rest of the conversation or end the turn.

## Scripts you write must report progress

The script owns the percentage; huginn only reads it. Print to stdout lines of exactly this form at meaningful checkpoints (per batch, file or step), not every iteration:

```
PROGRESS <0-100> [optional message]
```

Under huginn, stdout is a file, so most languages buffer it. In Python use `print(..., flush=True)` or run `python3 -u`; otherwise the PROGRESS lines appear only when the script ends and the user gets no notifications.

```python
for i, item in enumerate(items, 1):
    process(item)
    print(f"PROGRESS {i * 100 // len(items)} processed {i}/{len(items)}", flush=True)
```

Without any PROGRESS line huginn only sends a "still running, N min elapsed" ping every 10 minutes, with no ETA.

## Resume when the user comes back

Check the state yourself; do not ask the user to paste anything.

- `huginn status`: each run with its state (`RUNNING` / `done exit=N` / `dead`), run id and log path.
- Run state is in `~/.local/state/huginn/<run id>/`: `log` (full stdout+stderr) and `meta` (`desc`, `cmd`, `started`, `pid`, `finished`, `exit`, `goal`).

Read `meta` and the end of `log`, tell the user whether it succeeded, and continue the original goal (`goal` in `meta`).

## Cancel and clean up

- `huginn stop <run id>` sends SIGTERM to the command and its whole process group and records `exit=stopped`. Never `kill` the pid from `meta`: it is the supervisor, and killing it alone leaves the command running.
- `huginn prune [days]` deletes finished runs older than N days (default 30). Ask the user before pruning.

## Limits

- Jobs do not survive a reboot and are not restarted. Durable jobs need a systemd user unit.
- Notifications need a graphical session; over SSH or headless they fail silently, but the log and `huginn status` still work.

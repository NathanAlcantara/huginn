# Hermes — long-running background scripts

When I (Kiro) create a script or command that is likely to run longer than ~10 minutes, I run it through `hermes` instead of blocking the chat waiting for it. Hermes runs the command detached in the background and notifies the user via GNOME (`notify-send`) as it progresses, so the user can leave the chat, get pinged at milestones, and come back when it finishes without spending tokens on me polling progress.

`hermes` lives at `~/.local/bin/hermes` (bash, no dependencies beyond coreutils + `notify-send` + `setsid`).

## When to use it

- Any script/command I expect to take more than ~10 minutes (long builds, data processing, backfills, large downloads, batch jobs, big test suites).
- Only for commands that run and exit. Do not use it for interactive commands or long-lived servers.
- For quick commands, run them normally.

## How to run it

```
hermes "<short description>" <command> [args...]
```

Hermes launches the command, prints the PID and the log path, and returns immediately. I do NOT wait on it or poll it in the chat. After launching, I tell the user it is running in the background and that they will be notified, then I continue with whatever else the conversation needs.

## Always emit progress

Any long script I write MUST report its own progress so hermes can compute a real ETA. The script owns the percentage; hermes only reads it. Print lines to stdout in exactly this form:

```
PROGRESS <0-100> [optional message]
```

Example (bash loop over N items):

```bash
total=$#; i=0
for item in "$@"; do
  process "$item"
  i=$((i+1))
  echo "PROGRESS $(( i * 100 / total )) processed $i/$total"
done
echo "PROGRESS 100 done"
```

Emit `PROGRESS` at meaningful checkpoints (per batch/file/step), not every tiny iteration. Hermes notifies the user every 15% with the ETA derived from elapsed time and the reported percent. If a script emits no `PROGRESS` at all, hermes falls back to a plain "still running, N min elapsed" heartbeat every 10 minutes with no ETA — so always emit progress when I can.

## Resuming (manual loop)

The user drives the loop manually: when they come back and ask about a background job, I check its state myself — I do not need them to paste anything.

- `hermes status` lists recent runs with their state (RUNNING / done exit=N / dead) and log path.
- Each run stores state in `~/.hermes/<timestamp>-<slug>/`:
  - `log` — the command's full stdout+stderr (read this for results or errors).
  - `meta` — `desc`, `cmd`, `started`, `pid`, `finished`, `exit`, and optional `goal`.
- To record what I was trying to accomplish so I can resume the larger goal later, set `HERMES_GOAL="..."` in the environment when launching; it is saved into `meta`.

When the user returns, I run `hermes status`, read the relevant `meta` and `log`, determine whether the job succeeded, and continue the original objective of the conversation from there.

## Known limitation

Hermes uses `setsid` to detach, so jobs survive the Kiro session but NOT a machine reboot — there is no auto-restart. This is intentional and fine for one-off long runs. If durable, restartable jobs are ever needed, that calls for a real supervisor (systemd user unit), not hermes.

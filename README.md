# ghul-dashboard

Live view of what is happening across the ghūl workspace: Claude Code sessions
and the worktree each is working in, ghul-test runs with progress, and open
`claude/*` pull requests with CI and review state.

Written in ghūl. Run it in a spare terminal pane:

```sh
cd ghul-dashboard
dotnet run                      # 1s refresh; q, Escape or Ctrl-C quits
dotnet run -- --interval 2      # slower refresh
dotnet run -- --once            # one frame to stdout, for scripting
```

The workspace is found by walking up from the current directory for a directory
holding both `CLAUDE.md` and `worktrees/`; `GHUL_WORKSPACE` overrides.

## Where the data comes from

| Panel | Source |
| --- | --- |
| sessions | `~/.claude/sessions/<pid>.json`, filtered to live processes |
| session worktree | the owning run's directory, else the last `cwd` in the session's transcript |
| test runs | `$XDG_RUNTIME_DIR/ghul-test/<pid>.json`, written by ghul-test |
| worktrees | `git worktree list --porcelain` per checkout |
| pull requests | one `gh api graphql` search, refreshed every 120s |

A session's own `cwd` is where it was launched, which for this workspace is the
umbrella directory rather than a worktree, so the worktree column is derived two
ways. A session with a test run in flight is attributed exactly, by walking
`/proc/<pid>/stat` parent pids up from the run to a known session. Otherwise the
last `cwd` recorded in its transcript is the best available answer.

Run status files are reaped here as well as by the status line: a file whose
process is gone and which never reached `done` was abandoned by a killed run.

## Known limitations

- A `SIGTERM` or `SIGKILL` leaves the terminal on the alternate screen with the
  cursor hidden, since neither is handled. `reset` restores it. `q`, Escape and
  Ctrl-C all exit cleanly.
- Column widths are fixed rather than fitted to the terminal, so a narrow window
  wraps.
- The pull-request query covers checkouts directly under the workspace, and only
  branches named `claude/*`.

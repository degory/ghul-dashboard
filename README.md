# ghul-dashboard

Live view of what is happening across the ghūl workspace: machine load, Claude
Code sessions and the worktree each is working in, ghul-test runs with progress,
and open `claude/*` pull requests with CI and review state.

Written in ghūl. Run it in a spare terminal pane:

```sh
cd ghul-dashboard
dotnet run                      # 1s refresh
dotnet run -- --interval 2      # slower refresh
dotnet run -- --once            # one frame to stdout, for scripting
```

The workspace is the directory holding both `CLAUDE.md` and `worktrees/`, found
by walking up from the current directory and then from the directory this program
was built into — so running it from an unrelated shell works wherever that shell
happens to be. `GHUL_WORKSPACE` overrides both.

## Keys

| Key | |
| --- | --- |
| up / down | move the session selection |
| `k` | terminate the selected session — asks `y` / `n` first |
| Ctrl-L | repaint from a cleared screen |
| `q`, Escape, Ctrl-C | quit |

`k` sends `SIGTERM`, not a kill, so the session can save its transcript and exit
cleanly. It refuses to terminate a session that is an ancestor of this process,
which would take the dashboard down with it — that only arises when a session
spawned the dashboard itself; run from its own shell, no session owns it and the
confirmation is the only guard.

Selection follows a pid rather than a row, so rows reordering under it — as
sessions become active — never moves the selection to a different session.

## Layout

The frame is fitted to the terminal every refresh. Width is divided between the
two elastic columns, a session's name and its worktree, after the fixed columns
have taken what they need; the load meters shrink on a narrow window rather than
wrapping. Height is divided between the panels before any of them renders, so a
long session list cannot push the pull requests off the bottom — each panel that
has to truncate says how many rows it dropped. A resize clears the screen before
repainting, since the new layout does not necessarily cover what the old one drew.

## Structure

Parsing is kept apart from reading, which is what makes most of the program
testable without a machine to inspect:

| | |
| --- | --- |
| `text.ghul`, `proc-parse.ghul`, `worktree-parse.ghul`, `pull-request-parse.ghul` | pure functions from a file's or command's output to data |
| `files.ghul`, `process-runner.ghul`, `process-tree.ghul` | the only code that reads files or spawns processes |
| `*-source.ghul` | one per data source: reads, parses, and holds whatever state that source needs between frames |
| `collector.ghul` | starts the sources and assembles one snapshot |
| `layout.ghul` | the arithmetic that fits the frame to the terminal |
| `format.ghul`, `screen.ghul`, `renderer.ghul` | turning a snapshot into lines, and lines into a painted frame |

Each row's fixed cost is stated in `LAYOUT` as its parts rather than as a total,
so adding a column cannot leave a stale number behind. `LINE_BUILDER` then drops
any segment that would not fit, so a row cannot overrun its width even if the
arithmetic is wrong.

## Tests

```sh
dotnet test tests
```

73 tests over the parsers, the layout arithmetic, the formatters and the
selection behaviour — every part that is a function of its input rather than of
the machine. The awkward cases are the point: a `/proc/<pid>/stat` command
containing spaces and parentheses, counters that go backwards when a core is
hotplugged, a selection whose session has exited, a row at every width from 40
columns up.

There are no integration tests. Covering the rest would mean faking `/proc`, a
`~/.claude` directory and `git`/`gh`, and the seam that would make that possible
is the same parser split these tests already exercise.

## Where the data comes from

| Panel | Source |
| --- | --- |
| sessions | `~/.claude/sessions/<pid>.json`, filtered to live processes, most recently active first |
| session worktree | the owning run's directory, else the last `cwd` in the session's transcript |
| test runs | `$XDG_RUNTIME_DIR/ghul-test/<pid>.json`, written by ghul-test |
| worktrees | `git worktree list --porcelain` per checkout |
| pull requests | one `gh api graphql` search, refreshed every 120s |
| processor, memory | `/proc/stat`, `/proc/meminfo`, `/proc/loadavg` |
| busiest processes | `utime + stime` deltas from `/proc/<pid>/stat` |

A session's own `cwd` is where it was launched, which for this workspace is the
umbrella directory rather than a worktree, so the worktree column is derived two
ways. A session with a test run in flight is attributed exactly, by walking
`/proc/<pid>/stat` parent pids up from the run to a known session. Otherwise the
last `cwd` recorded in its transcript is the best available answer.

Every processor figure is a delta between two samples, so the monitor primes
itself at startup and enforces a minimum sampling window — a shorter one is
mostly noise, and a single frame of it reads as a spike. Per-process figures are
a share of one core, so a value above 100% means a process is spread across more
than one. `USER_HZ` is assumed to be 100, which holds on every Linux
configuration this runs on.

Run status files are reaped here as well as by the status line: a file whose
process is gone and which never reached `done` was abandoned by a killed run.

## Known limitations

- A `SIGTERM` or `SIGKILL` leaves the terminal on the alternate screen with the
  cursor hidden, since neither is handled. `reset` restores it. `q`, Escape and
  Ctrl-C all exit cleanly.
- The pull-request query covers checkouts directly under the workspace, and only
  branches named `claude/*`.
- Below about 60 columns the pull-request rows start dropping columns — the
  repository first, then the CI state — to keep the branch name readable.

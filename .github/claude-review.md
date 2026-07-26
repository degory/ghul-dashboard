# Cloud code review brief

What this repository is, and what to watch for in it. Everything else — what PR
context is available, how to post a review, what makes a finding worth raising,
comment hygiene, PR-description shape, the versioning mechanism — comes from the
review workflow's runtime notes. Don't restate it here: this file is read first,
so a stale copy would silently override the current text.

Not loaded by local Claude Code; only the cloud reviewer reads this.

## What this repo is

`ghul-dashboard` is a terminal UI giving a live view across the ghūl workspace:
machine load, Claude Code sessions and the worktree each is working in, `ghul-test`
runs with progress, and open `claude/*` pull requests with CI and review state.
Written in ghūl, run in a spare pane. It finds the workspace by walking up for a
directory holding both `CLAUDE.md` and `worktrees/`, with `GHUL_WORKSPACE`
overriding.

It is a read-only observer of a workspace that other sessions are actively working
in.

## What to watch for here

- **Writes or side effects.** The dashboard observes; it should not mutate git
  state, touch worktrees, or write anywhere but its own config. Flag any new write
  path on sight.
- **Blocking the render loop.** It redraws on a timer, so a synchronous network or
  subprocess call on the render path stalls the whole UI. Those belong off the
  critical path with a stale-value fallback.
- **Parsing external state.** It reads progress JSON, `git` output and the GitHub
  API, all of which can be absent, partial or half-written. Missing or malformed
  input must degrade to "unknown", never crash the frame.
- **Terminal handling.** Escape sequences, resize, and cursor restoration on exit -
  a crash leaving the terminal in raw mode is a genuine nuisance.

## Versioning

This repo publishes nothing. Version bumps are not a concern here.

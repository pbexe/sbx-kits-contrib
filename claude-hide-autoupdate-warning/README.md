# claude-hide-autoupdate-warning

A mixin for the built-in `claude` agent that hides Claude Code's **"unable to
auto-update" / "couldn't auto-update"** warning.

Inside a sandbox the Claude Code binary usually can't rewrite itself (it's baked
into a read-only base image), so every session nags that it failed to
auto-update. That update would be pointless here anyway — the version is pinned
by the image. This kit turns the auto-updater off, so the warning never shows.

## Quick start

Pair the mixin with the `claude` agent via `--kit`:

```console
$ sbx run claude --kit ./claude-hide-autoupdate-warning .
```

Or pull it straight from this repo (pinned by ref):

```console
$ sbx run claude --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=claude-hide-autoupdate-warning" .
```

## How it works

The auto-updater is disabled two ways, both no-ops if the other already took
effect:

- **`DISABLE_AUTOUPDATER=1`** is set via `environment.variables`. This is the
  authoritative switch — Claude Code honours it reliably and skips the update
  check entirely, which is what suppresses the warning.
- **`autoUpdaterStatus: "disabled"`** is merged into `~/.claude/settings.json`
  by the `install` hook (run as root) using `jq`. It creates the file if
  missing and leaves every other key untouched, then `chown`s `~/.claude` and
  `settings.json` back to the `agent` user. The chown is deliberately *not*
  recursive — read-only mounts are commonly bound under `~/.claude` (such as
  `~/.claude/skills`), and a `chown -R` would fail on them and abort the hook.
  The temp file is created inside `~/.claude` so the final `mv` is an atomic
  same-filesystem rename. Re-running is idempotent.

The settings key is a belt-and-suspenders fallback — it has been reported
ignored in some Claude Code versions
([anthropics/claude-code#13213](https://github.com/anthropics/claude-code/issues/13213)),
which is why the environment variable is the primary mechanism.

## Requirements

The `install` merge uses `jq`, which is present on the `claude-code` base image.
This mixin only makes sense on the `claude` agent, so it declares
`requires.agent: claude` and will error if composed onto any other base agent.

## Cleanup

The kit writes only to the sandbox (an env var and `~/.claude/settings.json`
inside the container). It creates no state on the host, so there is nothing to
clean up.

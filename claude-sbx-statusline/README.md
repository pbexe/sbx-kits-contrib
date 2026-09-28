# claude-sbx-statusline

A [Claude Code status line](https://docs.claude.com/en/docs/claude-code/statusline) for Docker
Sandboxes, built on [claude-powerline](https://powerline.owloops.com/)
([`@owloops/claude-powerline`](https://github.com/Owloops/claude-powerline)). Compose this mixin
onto the built-in `claude` agent and every session renders a vim-style powerline bar of where you
are and what the session is costing.

## What you get

A single powerline row in the `tokyo-night` theme, with these segments:

| Segment | Shows |
| --- | --- |
| directory | current directory (basename only) |
| git | branch, with a dirty marker |
| model | active model |
| session | tokens used this session |
| today | today's cost, against a $50 budget |
| block | current 5-hour block cost + burn rate, against a $15 budget |
| context | context-window usage (33k autocompact buffer reserved) |
| agent | active agent |

Budget segments turn red past 80% of their threshold. The `weekly`, `version`, `tmux`,
`sessionId`, `metrics`, `thinking`, `cacheTimer` and `outputStyle` segments ship disabled —
flip `enabled` to `true` in the config to add them.

## Quick start

Pair the mixin with the `claude` agent via `--kit`:

```console
$ sbx run claude --kit ./claude-sbx-statusline .
```

Or pull it straight from this repo (pinned by ref):

```console
$ sbx run claude --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=claude-sbx-statusline" .
```

## Configuration

The kit ships its config to **`~/.claude/claude-powerline.json`** — the user-level path
claude-powerline looks for. Edit it in a running sandbox to change theme, display style, segments,
or budgets; the next render picks the change up, no restart needed.

claude-powerline searches, in priority order:

1. `./.claude-powerline.json` (per-project — overrides the kit's file)
2. `~/.claude/claude-powerline.json` (what this kit writes)
3. `~/.config/claude-powerline/config.json`

`CLAUDE_POWERLINE_THEME`, `CLAUDE_POWERLINE_STYLE` and `CLAUDE_POWERLINE_CONFIG` override the
config file; CLI flags override those. Themes: `dark`, `light`, `nord`, `tokyo-night`,
`rose-pine`, `gruvbox`, `custom`. Styles: `minimal`, `powerline`, `capsule`, `tui`.

> [!NOTE]
> The config sets `"charset": "unicode"`, so the segment separators and icons need a
> [Nerd Font](https://www.nerdfonts.com/) in **your terminal** (the font is a host-side thing —
> the sandbox can't supply it). Without one you'll see tofu boxes; set `"charset": "text"` for
> ASCII-only output instead.

## How it works

- **The `install` hook** (run as root) does two things:

  1. `npm install -g @owloops/claude-powerline@latest`. Upstream suggests `npx -y …` as the
     status line command, but that re-resolves the package on every render; a global install
     makes each render a local exec. It also means the status line keeps working if npm is
     unreachable later — rendering itself reads local session files and needs no network.
  2. Merges the `statusLine` block into `~/.claude/settings.json` with `jq`:

     ```json
     {
       "statusLine": { "type": "command", "command": "/usr/local/bin/claude-powerline" }
     }
     ```

     The exact path is whatever `command -v claude-powerline` resolves to at install time —
     root's global npm prefix isn't necessarily the agent's, so the absolute path is recorded
     rather than trusting the bare name to be on the agent's `PATH`. No `--style` flag is
     passed, so `claude-powerline.json` stays the single source of truth (a CLI flag would
     silently outrank anything you edit there).

  The hook creates `settings.json` if missing and leaves every other key untouched — only
  `statusLine` is set (replacing a prior one if present) — then `chown`s `~/.claude` and
  `settings.json` back to the `agent` user. The chown is deliberately *not* recursive: read-only
  mounts are commonly bound under `~/.claude` (such as `~/.claude/skills`), and a `chown -R` would
  fail on them and abort the hook. Re-running is idempotent. The temp file is created inside
  `~/.claude` so the final `mv` is an atomic same-filesystem rename rather than a cross-device copy.
- **`files/home/.claude/claude-powerline.json`** is copied to `~/.claude/claude-powerline.json`
  at sandbox start.

## Requirements

`node` (≥ 18), `npm` and `jq` — all present on the `claude-code` base image. The install hook
needs `registry.npmjs.org` reachable, which the kit's `caps.network` block requests.

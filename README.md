# claude-ez

A private Claude Code marketplace carrying one plugin, `ez` — nine skills that
take a change from a planned direction through critical review to open pull
request, and tidy the checkout once it merges, in any git repository and any
language. `plugins/ez/README.md`
documents the chain itself and the per-repository `.claude/ez.json` schema.

```
.claude-plugin/marketplace.json
plugins/ez/                      the plugin; see its own README
```

## Install on a device

The marketplace clones over SSH, so the key for this repository has to be loaded
in `ssh-agent`. Use SSH rather than HTTPS: background refreshes run with git
credential helpers disabled, so a private HTTPS marketplace stops updating
silently, while an SSH one keeps working.

```sh
claude plugin marketplace add git@github.com:coffeebeats/claude-ez.git
claude plugin install ez@claude-ez
```

Verify with `claude plugin details ez` — nine skills, seven agents, no hooks.
`/ez:plan`, `/ez:review`, `/ez:fix`, `/ez:comments`, `/ez:commit`, `/ez:pr`,
`/ez:publish`, `/ez:clean`, and `/ez:no-ai-slop` are then available everywhere.

Turn on auto-update, which is off by default for a marketplace Anthropic does
not run and which adding the marketplace never turns on: set `"autoUpdate":
true` on the `claude-ez` entry under `extraKnownMarketplaces` in
`~/.claude/settings.json`, or use `/plugin` → Marketplaces. Without it, a push
to `main` reaches the device only through `claude plugin update ez`.

Set `CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` so a failed background
pull keeps the existing clone instead of deleting and re-cloning it.

## Working on the plugin

The machine where the plugin is edited installs from the remote like any other.
To try an edit, start a session from this checkout with the working tree loaded
over the installed copy:

```sh
claude --plugin-dir ./plugins/ez
```

That session loads `plugins/ez/` in place as `ez@inline`, which replaces
`ez@claude-ez` for that session alone, and `/reload-plugins` picks up each
further edit without a restart. No other session sees the working tree.

**Never add this checkout as the marketplace.** A marketplace added from a local
folder also loads its plugins in place, but for every session on the machine, in
every repository and at every scope, so a half-finished edit is what the next
session anywhere runs.

An edit reaches the installed copy only once it is on `main`:

```sh
git push origin main
claude plugin update ez   # copies the new commit into the plugin cache now
```

With auto-update on, the update also runs by itself in each interactive
session, within ten minutes of its first message. Either way it rewrites the
cache, not the sessions already running: each keeps the copy it loaded until
`/reload-plugins` is run in it or it restarts.

Nothing may live in `~/.claude/skills/ez/`. A directory there loads as a second
copy of the same plugin under `ez@skills-dir`, and both sets of skills are then
offered at once.

## Licence

MIT, in `LICENSE`. The one exception is `plugins/ez/skills/no-ai-slop/`, which is
vendored from a third party under its own MIT notice — that directory keeps its
`LICENSE` and `UPSTREAM.md`, and both must travel with any copy of it.

## Versioning

`plugin.json` and the marketplace entry deliberately carry no `version`. With
none set, Claude Code resolves the plugin to its commit SHA, so
`claude plugin update ez` and auto-update track the branch, and a running
session picks up the new copy at `/reload-plugins` or its next start. Adding a
version means bumping it on every release or clients silently keep serving the
cached copy.

## Turning it off in one repository

A repo with its own version of these skills should silence this plugin rather
than have both offered. From inside that repo:

```sh
claude plugin disable ez@claude-ez --scope local
```

That writes `enabledPlugins` into the repo's `.claude/settings.local.json`, which
Claude Code keeps out of git. `skillOverrides` does not apply to plugin skills.

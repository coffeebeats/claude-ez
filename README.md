# claude-ez

A private Claude Code marketplace carrying one plugin, `ez` — four skills that
take a change from working tree to open pull request in any git repository and
any language. `plugins/ez/README.md` documents the chain itself and the
per-repository `.claude/ez.json` schema.

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

Verify with `claude plugin details ez` — four skills, no hooks. `/ez:commit`,
`/ez:pr`, `/ez:comments`, and `/ez:no-ai-slop` are then available everywhere.

Set `CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` so a failed background
pull keeps the existing clone instead of deleting and re-cloning it.

## Working on the plugin

On the machine where the plugin is edited, point the marketplace at this
checkout instead of at the remote, which drops the network from the edit loop.

```sh
claude plugin marketplace add ~/projects/coffeebeats/claude-ez
claude plugin install ez@claude-ez
```

Only one of the two can be registered at a time; they share the marketplace name
`claude-ez`. Remove the other first with `claude plugin marketplace remove
claude-ez`.

**An edit to the working tree is not live.** Installing copies the plugin into
`~/.claude/plugins/cache/claude-ez/ez/<version>/`, and that copy is what every
session loads. For a directory source backed by a git repository the version is
the resolved commit SHA, so the loop is:

```sh
git commit ...            # the SHA changes, which is the only refresh trigger
claude plugin update ez   # re-copies into a new cache directory
                          # then restart Claude Code
```

`claude plugin update` compares **versions, not content**. An uncommitted edit
leaves the SHA unchanged, so update reports "already at the latest version" and
the stale copy keeps being served with no warning. If a cache entry is ever
wrong, `claude plugin uninstall ez@claude-ez` followed by a fresh install
rebuilds it; deleting the cache directory alone does not, because the recorded
`installPath` then points at nothing.

Nothing may live in `~/.claude/skills/ez/`. A directory there loads as a second
copy of the same plugin under `ez@skills-dir`, and both sets of skills are then
offered at once.

## Versioning

`plugin.json` and the marketplace entry deliberately carry no `version`. With
none set, Claude Code resolves the plugin to its commit SHA, so
`claude plugin update ez` tracks the branch. Adding a version means bumping it
on every release or clients silently keep serving the cached copy.

## Turning it off in one repository

A repo with its own version of these skills should silence this plugin rather
than have both offered. From inside that repo:

```sh
claude plugin disable ez@claude-ez --scope local
```

That writes `enabledPlugins` into the repo's `.claude/settings.local.json`, which
Claude Code keeps out of git. `skillOverrides` does not apply to plugin skills.

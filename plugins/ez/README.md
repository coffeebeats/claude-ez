# ez — a portable commit chain for Claude Code

Four skills that carry a change from working tree to open pull request, in any
git repository and any language. Nothing here is language-specific; per-repository
detail lives in that repo's own `.claude/ez.json` and `.claude/rules/comments.md`.

```
.claude-plugin/plugin.json
skills/comments/          audit and trim the comments in changed files
skills/commit/            land the tree as ordered atomic commits
skills/pr/                push the branch and open the PR
skills/no-ai-slop/        prose judgment; vendored, see UPSTREAM.md
```

## The chain

Every step is user-invoked; the chain never advances on its own.

1. `/ez:comments [files]` — audits and rewrites comments in the changed files to
   the house style. Defaults to everything dirty against `HEAD`.
2. `/ez:commit` — reviews the tree and lands it as an ordered series of one-line
   conventional commits, one slice at a time. Pass `manual` to have it stage each
   slice and hand the commit back.
3. `/ez:pr [target]` — pushes the branch and opens a PR titled as a one-line
   conventional commit. Pass `manual` to approve the title and body before
   anything reaches the remote.

`/ez:no-ai-slop` sits under the chain rather than in it. It owns prose judgment —
PR titles and bodies, docs, and the wording inside comments — and steps 1 and 3
invoke it themselves. Invoke it directly to edit any prose, or with "is this
slop?" to audit without rewriting.

Nothing here is enforced by a hook. `/ez:commit` is the way to commit because
each skill says so and because its description says so in every session, not
because a guard denies the alternative.

**A project's own version of any of these wins.** A repo carrying
`.claude/skills/*commit*/`, `*pr*/`, `*comment*/`, or its own prose skill knows
the house conventions; each skill checks for one and defers. Plugin skills are
namespaced, so a repo's `no-ai-slop` and this one coexist without shadowing.

## Per-repository configuration

`.claude/ez.json` at a repo root tunes the chain. Every key is optional, and the
skills detect whatever it omits.

```json
{
  "gate": { "fast": ["<format>", "<lint>"], "full": ["<tests>"] },
  "commit": {
    "types": ["feat", "fix", "refactor", "chore"],
    "scopes": ["<scope>"],
    "protectedBranches": ["main"],
    "branchPattern": "<type>/<short-desc>"
  },
  "comments": { "rules": ".claude/rules/comments.md", "lint": "<command>" },
  "pr": { "footer": "" }
}
```

`protectedBranches` tells `/ez:commit` which branches to auto-branch away from
before staging anything.

## Install

Requires `gh` authenticated for `/ez:pr`.

```sh
claude plugin marketplace add git@github.com:coffeebeats/claude-ez.git
claude plugin install ez@claude-ez
```

Verify with `claude plugin details ez`, which should list four skills and no
hooks.

## Turning it off in one repository

A repo with its own version of these skills should silence this plugin rather
than have both offered. From inside that repo:

```sh
claude plugin disable ez@claude-ez --scope local
```

That writes `enabledPlugins` into the repo's `.claude/settings.local.json`, which
Claude Code keeps out of git. `skillOverrides` does not apply to plugin skills.

## A note on versioning

`plugin.json` deliberately carries no `version`. With none set, Claude Code
resolves the plugin to its commit SHA, so `claude plugin update ez` tracks the
branch and a stale copy is never served from cache. Adding a version means
bumping it on every release or clients silently keep the old copy.

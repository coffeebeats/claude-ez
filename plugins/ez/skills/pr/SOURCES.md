# Why `/ez:pr` handles stacks the way it does

The stack rules in `SKILL.md` encode how `gh stack` behaves, read from its
source rather than its help text, which says less. Read against
`github/gh-stack` v0.1.1 in October 2026. After `gh extension upgrade stack`,
re-check each point below before trusting the skill. Paths are relative to
<https://github.com/github/gh-stack/tree/v0.1.1>.

## `link`, not `submit`

`submit` pushes every branch with `--force-with-lease` (`cmd/submit.go:227`,
`git.Push(..., true, false)`), which the no-force-push rule forbids. Run
outside a terminal it also titles new PRs itself. `link` exists for branches
managed by another tool, which is what the ez chain is, and pushes only the
branch names it is handed, with a plain push (`cmd/link.go:430`). Given a
branch with no PR, it opens one with a generated title (`createMissingPRs`,
`cmd/link.go:640`). That is why the skill opens every PR itself and hands
`link` only PR numbers.

## `link`'s two forms

- **First argument a stack number** (`runLinkAdd`, `cmd/link.go:255`). The rest
  are appended to the top, PRs already in the stack are skipped, `--base` is
  ignored, and each appended PR's base is set to the branch below it, starting
  from the stack's top branch. This is the only form that adds to an existing
  stack without naming every PR in it.
- **Only PR numbers or branches** (`runLinkCreateOrUpdate`, `cmd/link.go:179`).
  The arguments are the whole stack, bottom first. If they match an existing
  stack that holds a PR they leave out, `prevalidateStack` (`:607`) refuses
  before anything changes. Otherwise `fixBaseBranches` (`:681`) sets the bottom
  PR's base to `--base`, falling back to the repo's default branch, and every
  other PR's base to the branch below it. So two numbers from the middle of an
  unlinked chain would move the lower PR onto `main`. The skill always passes
  `--base` and the whole chain in this form.

A stack keeps its merged PRs. `syncStack` sends them because the API rejects a
list without them (`cmd/submit.go:705`). So the whole-chain form cannot extend
a stack once its bottom has merged. Appending by stack number can.

## Finding a PR's stack

`GET repos/{owner}/{repo}/stacks?pull_request=<n>` returns the one stack
holding that PR, with its number and its PRs bottom first, each with
`merged_at` and its head branch (`FindStackForPR`,
`internal/github/github.go:515`). Stack numbers come from the same counter as
PR and issue numbers, so a stack can take the number a PR would have had.

## What `gh stack view --json` knows

`runViewJSON` refreshes PR state before printing (`syncStackPRs`,
`cmd/utils.go:479`). An untracked branch picks up an *open* PR found by its
name, and a tracked PR is re-read by number, so `isMerged`
(`internal/stack/stack.go:99`) is live for any PR `gh stack` has seen open.
A PR that merged before any `view` or `sync` saw it is never found, which is
why the skill checks the open PRs too. Offline, the refresh fails quietly and
the stored state prints.

`needsRebase` tests whether the nearest unmerged branch below, or the trunk
for the bottom entry, is an ancestor of the branch (`ActiveBaseBranch`,
`internal/stack/stack.go:172`; `cmd/view.go:250`). It fires when that branch
or the trunk only moved ahead, and it misses a squash-merged branch once
`isMerged` skips it, so the skill doesn't use it. Each entry's `base` is the
parent's tip when the branch was cut or last restacked. The skill instead
checks that this commit is still in the target's history, which fails only
after a rewrite or a squash merge.

## Living with the user's own `gh stack` commands

A stack `/ez:commit` builds with `init` and `add` has no remote ID. When the
user later runs `gh stack sync`, `reconcileRemoteStack` (`cmd/utils.go:1555`)
skips the remote stack check, and `syncStack` falls through to
`reconcileUntrackedStack` (`cmd/submit.go:739`), which adopts a matching stack
that `link` created instead of creating a second one. `add` works only from
the top of a stack (`cmd/add.go:96`), so `/ez:commit` falls back to
`git switch -c` anywhere else.

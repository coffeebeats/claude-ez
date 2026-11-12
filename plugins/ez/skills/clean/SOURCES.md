# Why `/ez:clean` is shaped the way it is

`/ez:clean` deletes things, so most of its design is about what it may delete
without asking. The git claims below were run against git 2.45 in a scratch
repository in November 2026. The `gh stack` claims were read from
`github/gh-stack` v0.1.1 source, with paths relative to
<https://github.com/github/gh-stack/tree/v0.1.1>. The Claude Code claims come
from its documentation as fetched the same month. Re-check each after an
upgrade.

## Evidence instead of a prompt

Anthropic's skill authoring guide sorts instructions by "degrees of freedom"
and puts fragile, error-prone operations at the low end, with exact commands
and no latitude
(<https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices>).
It also recommends a plan, validate, execute sequence for destructive changes.
The skill follows both. Phase 1 classifies and writes nothing, Phase 3 prints
the plan, and Phase 4 runs fixed commands in a fixed order.

The plan is not a gate. The chain already dropped the approval stops the user
always approved, and a deletion backed by proof that the work is elsewhere is
that kind of stop. One question remains, about untracked files the session did
not create and a `picked` plan that looks built and landed: its branch is
finished and commits on the default branch have touched its files since. No
command can tell whether an untracked file is wanted, and deleting it has no
undo. Nor can one tell whether those commits built the plan: `/ez:plan` leaves
a plan `picked` both before it is built and after a build that missed a
criterion, so the heading cannot settle it. A state file that may describe
those untracked files rides in that question rather than adding a second one;
see **State files**. A landed worktree that holds ignored files rides in the
same question: removing it deletes them, and no command tells a cache that
costs a rebuild from a `.env` that cannot be rebuilt.

## Finding the default branch

A remote added with `git remote add` and then fetched has no
`refs/remotes/<remote>/HEAD`; a clone sets it, and on git 2.45 a fetch does
not. In a scratch repository whose remote's default was `trunk`,
`git symbolic-ref` then exited 128, and
`git ls-remote --symref origin HEAD` printed a `ref: refs/heads/trunk` line
and left every local ref as it was. That is why the hard rules allow it
beside `git fetch`: it reads the remote and writes nothing. It exited 128
against an unreachable remote and printed nothing against an empty one, and
the skill stops on either rather than guess `main` or `master`. With the real
default `trunk` and an `origin/main` beside it, the guess would leave the
local `trunk` unprotected, and the Ancestor proof clears it whenever
`origin/main` contains it.

## What counts as landed

`git branch -d` cannot be the test. On a squash-merged branch it prints
"error: the branch 'feat/sq' is not fully merged", because the squash commit
shares no history with the branch. `git cherry <default> <branch>` marks every
commit `+` for the same reason.

An upstream marked `[gone]` after `git fetch --prune` cannot be the test
either. GitHub deletes the head branch when a PR merges, when the repo setting
is on, and a user deletes it when closing a PR unmerged. Both leave the same
`[gone]`.

The four proofs, in the order the skill tries them:

- **Ancestor.** `git merge-base --is-ancestor` covers merge commits and
  fast-forwards. A branch with no commits of its own passes too. Deleting it
  loses a name and no work.
- **Merged PR at this tip.** `gh pr list --head <branch> --state merged`
  returns `headRefOid`, the commit the PR held when it merged. Comparing it to
  the local tip separates the branch that merged from a later branch that
  reused the name, and from a branch with commits added after the merge.
  `/ez:pr` matches PRs to branches by `headRefOid` for the same reason.
- **Nothing left to merge.**
  `git merge-tree --write-tree <default> <branch>` printed the same tree as
  `<default>^{tree}` for a two-commit branch squash-merged as one commit. It
  needs no forge and no PR, which matters in a repository that lands work by
  pushing a squash to the default branch. `--write-tree` arrived in git 2.38.
  The test is conservative: if the default branch later reverted the squash,
  the merge would bring the change back, the trees differ, and the branch is
  kept by this proof.
- **Its diff landed as one commit.** With the branch squash-pushed as S and a
  later commit editing the same lines, `merge-tree` exited 1 on a conflict.
  `git diff-tree -p <base> <branch> | git patch-id --verbatim` printed the ID
  that `git rev-list <base>..<default> | git diff-tree --stdin -p | git patch-id --verbatim`
  printed beside S. git-patch-id(1) says `--stable` and `--unstable` ignore
  whitespace, and with `--stable` a branch that added only trailing spaces
  after S hashed to S's ID; with `--verbatim` it did not. `diff-tree` is
  plumbing, so `color.diff` and other `git diff` settings do not reach the
  hash: with `color.diff=always`, `git diff` piped to `patch-id` printed
  nothing. A git that rejects an option exits 129 with empty stdout, so a git
  without `--verbatim` fails the proof rather than passing it. This proof
  does pass a squash the default branch later reverted: S still holds the
  branch's exact diff, and the printed tip restores the branch.

A deleted branch can be restored. After `git branch -D`, running
`git branch <name> <sha>` with the SHA git printed brought the branch back.
That is why the report prints each tip.

## Leaving the branch

`git switch` carries tracked changes to the new branch when they do not
conflict. In the test it switched with a modified file and left it modified on
the destination. A change made on a feature branch would then sit in the
default branch's tree. So the skill stays put when tracked changes exist, and
does not stash them: a stash is easy to forget, and `/ez:commit` never stashes
live work either.

`git merge --ff-only` on a default branch that is ahead of the remote reports
"Already up to date" and changes nothing. It fails when the two have diverged.
The skill never resets to make it pass.

## Worktrees

Tested behaviour of `git worktree`:

- `remove` on a worktree with an untracked file fails with "contains modified
  or untracked files, use --force to delete it". The skill never passes
  `--force`, so git's own check backs the Phase 1 status check for those.
- `remove` on a worktree holding only an ignored `local.env` succeeded and the
  file was gone, and one holding an ignored `.env`, `node_modules/`, and
  `src/__pycache__/` was removed whole. Git's check does not cover ignored
  files, so Phase 1 runs
  `status --porcelain=v1 --ignored --untracked-files=normal`, which printed
  one `!!` line for each of those three, and sends a landed worktree that
  holds any to the Phase 2 question rather than keeping it on every run.
  Without `--untracked-files=normal`, the same status printed nothing under
  `status.showUntrackedFiles=no` and each file inside the two directories
  under `all`.
- `remove --force` on a locked worktree fails and asks for `-f -f` or an
  unlock. The skill treats a lock as a claim that the worktree is in use.
- `git branch -d` and `-D` both refuse a branch "used by worktree at …". That
  fixes the order: worktrees before branches.
- `prune --dry-run -v` reports a worktree whose directory was deleted by hand
  as "gitdir file points to non-existent location", and `list --porcelain`
  marks it `prunable`.
- `add` writes the new worktree's `HEAD` file under
  `<common dir>/worktrees/<id>/` when it creates it, and a plain `add` leaves
  no `locked` line in `list --porcelain`. A later `status` in the worktree left
  that file's modification time alone. So the skill keeps a worktree whose
  `HEAD` changed within the hour: a just-started agent's worktree is clean and
  on a branch with no commits, and nothing else marks it as in use. This is a
  stand-in for idleness. A clean worktree idle for over an hour is not covered.

Claude Code creates its worktrees "under `.claude/worktrees/<name>/` at your
repository root, on a new branch named `worktree-<name>`"
(<https://code.claude.com/docs/en/worktrees>). The same page says a subagent's
worktree "with changes stays on disk", and that non-interactive runs do not
clean theirs up. Those are the worktrees the skill expects to find. It applies
the general test and has no rule keyed to the path or the branch prefix, so a
worktree the user made by hand gets the same protection.

The skill does not remove the worktree it runs in. The session's working
directory would vanish under it, and the state files the run reads live in
that worktree's git directory.

## Untracked and ignored files

`git clean -nd` lists untracked files. Only `-X` or `-x` reaches ignored ones.
Ignored files are out of scope: a dependency directory or build cache is
expensive to rebuild, and an ignored `.env` cannot be rebuilt at all.

The skill deletes with `git clean -f -- <path>` and not `rm`. Handed a
modified tracked file beside an untracked one, `git clean` removed the
untracked file and left the tracked one modified, so a wrong path costs
nothing.

"Scratch" is defined by what the session did, not by a filename pattern. A
`*.log` or `tmp_*` rule would match a file the user keeps on purpose. A session
that wrote a file knows why. After a compaction or in a new session that
knowledge is gone, and the skill says to treat everything as unknown.

## State files

A state file headed with a protected branch was written there before the work
had a branch of its own, so its branch never lands and never goes away.
`/ez:review` on the default branch reviews the working tree, untracked files
included (`skills/review/SKILL.md`, **Inputs**, the no-target case), and
`/ez:commit` adopts a report headed with the branch it left, at the current
HEAD, as the review of the work it is about to commit
(`skills/commit/SKILL.md`, Phase 0 step 3). A clean report cites no path, an
`ez-commit-plan.md` has no fixed format, and `skills/plan/SKILL.md` lets a
plan's `Touches:` line name modules, so the paths a file cites cannot show
which untracked files it describes. The skill reads the tree instead. With no
tracked changes and nothing untracked but the session's scratch, which the
same run deletes, the work such a file described is committed or gone. With
Unknown files left, it cannot tell, and the file goes into the one question
with them. Keeping it whenever any untracked file exists would let one file
the user keeps hold it for good.

The staging patches need no such test. `/ez:commit` writes `ez.patch` with
`git diff -- <file>`, after `git add -N` for a new file, which git-add(1) says
places an entry for the path in the index, so the patch only ever describes
tracked changes (`skills/commit/SKILL.md`, **Staging mechanics**). `/ez:fix`
writes `ez-fix.patch`, applies it in the same step, applies the candidate by
hand whenever the patch cannot be used, and removes it after the last item
(`skills/fix/SKILL.md`, Phase 4 step 1).

## Stacks

`gh stack sync --prune` deletes local branches for merged PRs itself, with
`git branch -D` (`cmd/sync.go:330`, `DeleteBranch(name, true)`). It also
rebases the remaining branches and pushes them with `--force-with-lease`,
which the chain's rules leave to the user, so the skill does not run it.

Deleting a merged stack branch by hand is safe only once `gh stack` has
recorded the merge. `resolveOriginalRefs` (`cmd/utils.go:893`) skips a branch
that is `IsMerged()` and missing, and backfills its SHA from the stack file.
A missing branch it does not know to be merged goes to `git.RevParseMap` and
fails the next rebase. `gh stack view --json` refreshes PR state before
printing, per `skills/pr/SOURCES.md`, so the skill deletes a branch in a live
stack only when that output marks it `isMerged`.

`view` reads only the current branch's stack, so every other stack is read
from the state file: JSON named `gh-stack` (`internal/stack/stack.go:16`,
`stackFilePath` at `:309`), holding `stacks[].trunk` and
`stacks[].branches[]`, each with `branch` and `pullRequest.merged`
(`stack.go:21-48`, `:229-232`). `IsMerged()` reads that field
(`stack.go:99`). `gh stack` locates it with `git rev-parse --git-dir`
(`internal/git/gitops.go:110`, calling cli/cli v2.93.0 `git/client.go:675`),
which in a linked worktree is that worktree's own directory. Each worktree
keeps its own file while branches are shared, so the skill reads them all.

`gh stack unstack --local` removes the stack from the local file and never
contacts GitHub (`cmd/unstack.go`, the `opts.local` branch of
`unstackTrackedStack`). With no argument it acts on the stack holding the
current branch. Its one argument is a stack number, looked up in the local
file only under `--local` (`lookupStackByNumber` with `allowRemote` false).
`view --json` omits the number; `view --short` prints it as `Stack #<n>` when
the stack has one (`cmd/view.go`, `viewShort`). So the skill reads the number
on the branch and unstacks by it once the run is on the default branch: a run
left off the default branch keeps the record, and with it the `isMerged` rule
above. A stack with no
number, one never created on GitHub, keeps its record. A finished stack the
session is not standing in keeps its local record; the skill reads it but
does not drop it.

## Who can invoke it

The Claude Code skills page says to "use `disable-model-invocation: true` for
workflows with side effects or that you want to control timing"
(<https://code.claude.com/docs/en/skills>). The skill leaves it off, as
`/ez:publish` does. The user asked for cleanup to follow from saying the work
is done, without typing the command. The description and **Inputs** name the
requests that count, and say that opening a PR is not one.

The same page says `allowed-tools` "does not restrict which tools are
available"; it pre-approves them for the turn. It lists `Bash` unscoped, as
the sibling skills do. The page shows scoped forms such as `Bash(git add *)`
but gives no grammar for them, and the limits that matter here are in the hard
rules.

The page truncates a description at 1,536 characters in the skill listing and
advises putting the key use case first, so the description opens with when the
skill applies.

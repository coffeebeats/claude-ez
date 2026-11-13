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
same question, since removing it deletes them, unless every one is a
directory a tool provably generated; see **Generated directories**. Asking
about a `.godot/` or a `.venv/` on every cleanup was a stop the user always
approved.

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
  or untracked files, use --force to delete it". For a worktree without
  submodules the skill never passes `--force`, so git's own check backs the
  Phase 1 status check for those.
- `remove` on a worktree holding only an ignored `local.env` succeeded and the
  file was gone, and one holding an ignored `.env`, `node_modules/`, and
  `src/__pycache__/` was removed whole. Git's check does not cover ignored
  files, so Phase 1 runs
  `status --porcelain=v1 --ignored --untracked-files=normal`, which printed
  one `!!` line for each of those three, and sorts each entry by
  **Generated directories** rather than keeping the worktree on every run.
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

### Worktrees with submodules

The first version never forced, so it never removed a worktree with an
initialized submodule, and in a repository with submodules every landed
worktree stayed. Tested in a scratch superproject holding a submodule that
holds another, with each linked worktree made by `worktree add` and
`submodule update --init --recursive`:

- `remove` on a clean worktree failed with "working trees containing
  submodules cannot be moved or removed". Git checks for submodules before it
  checks for dirt, and its source says "Until we sort this out, all
  submodules are 'dirty'" (`builtin/worktree.c`, `check_clean_worktree` and
  `validate_no_submodules`, at
  <https://github.com/git/git/blob/v2.45.2/builtin/worktree.c>). A worktree
  counts as having submodules when its git directory holds a `modules/`
  directory or its index has a populated gitlink. An empty `modules/` alone
  made `remove` fail; after `rmdir` it succeeded. git-worktree(1) says
  "Unclean worktrees or ones with submodules can be removed with `--force`"
  (<https://git-scm.com/docs/git-worktree>). No unforced path exists.
- `remove -f` removed it, and `<common dir>/worktrees/<id>/`, which holds the
  submodules' repositories under `modules/` (the submodule's `.git` file read
  `gitdir: ../../sup/.git/worktrees/w1/modules/lib`), was gone with it.
- `remove -f` also removed a worktree with a modified tracked file: one `-f`
  skips git's dirty check along with the submodule check. That is why the
  skill runs the whole test again immediately before forcing.
- `remove -f` on a locked worktree with submodules failed and asked for
  `-f -f`. A lock taken after Phase 1 still holds.
- A local branch, a stash, and a commit in the nested submodule, with its
  HEAD back at the recorded commit, left the superproject's
  `status --porcelain=v1 --ignored --untracked-files=normal --ignore-submodules=none`
  empty. `git rev-list --all HEAD --not --remotes` in that submodule listed
  the branch's commit and the stash's two commits. A commit at a detached
  HEAD that no ref holds was listed too, because of `HEAD`.
- With `diff.ignoreSubmodules=all` set for every repository, as `-c` or a
  global config sets it, an untracked file in the nested submodule left the
  superproject's status empty with and without `--ignore-submodules=none`:
  the flag does not reach past the first level. The same status with the
  flag, run inside the nested submodule, printed `?? u`. So the skill runs
  status in every submodule `foreach --recursive` visits. With the setting
  only in the superproject's own config, the superproject's flagged status
  did print ` M lib`.
- A submodule cloned in place, with a `.git` directory inside the worktree,
  kept its repository there, and `rev-parse --absolute-git-dir` named it. A
  walk of `modules/` found only the stale repositories beside it, while
  `rev-list` in the real one listed its unique commit. That is why the proof
  runs in each visited submodule's own repository.
- A nested submodule whose checkout was emptied is not visited by
  `foreach --recursive`, but its repository under `modules/lib/modules/inner`
  remained with a unique commit, which `rev-list` there listed, and `-f`
  would delete it. `find <modules> -type d -name objects` found it, whose
  parent holds `HEAD`. So the proof also walks `modules/`.
- In a worktree whose submodules were never initialized, `modules/` did not
  exist, `foreach` printed nothing, and plain `remove` succeeded as before.

`submodule deinit` is no way around the flag. It deletes the
`submodule.<name>` section of the repository's config, which every worktree
shares (`builtin/submodule--helper.c` and `path.c` `common_list` at
<https://github.com/git/git/tree/v2.45.2>). Run with `-f --all` in a linked
worktree, it took the main checkout's two `submodule.*` config lines to zero,
and `git submodule status` there marked `lib` uninitialized. Plain `remove`
on that worktree still refused afterwards.

The skill does not remove the worktree it runs in. The session's working
directory would vanish under it, and the state files the run reads live in
that worktree's git directory.

## Untracked and ignored files

`git clean -nd` lists untracked files. Only `-X` or `-x` reaches ignored ones.
The checkout's own ignored files are out of scope: a dependency directory or
build cache is expensive to rebuild, and an ignored `.env` cannot be rebuilt
at all. A landed worktree's ignored files go with the worktree, under the
rule in **Generated directories**.

The skill deletes with `git clean -f -- <path>` and not `rm`. Handed a
modified tracked file beside an untracked one, `git clean` removed the
untracked file and left the tracked one modified, so a wrong path costs
nothing.

"Scratch" is defined by what the session did, not by a filename pattern. A
`*.log` or `tmp_*` rule would match a file the user keeps on purpose. A session
that wrote a file knows why. After a compaction or in a new session that
knowledge is gone, and the skill says to treat everything as unknown.

## Generated directories

git-clean(1) says `-X` removes "only files ignored by Git", which "may be
useful to rebuild everything from scratch, but keep manually created files"
(<https://git-scm.com/docs/git-clean>). Git has no finer notion: an ignored
`.env` and an ignored `node_modules/` look the same to it. So the skill reads
evidence off the entry itself, and an entry with none is asked about as
before. A wrong Unknown costs one question; a wrong Generated costs data, so
every test leans to Unknown.

Only a directory can be Generated. A tool's cache or environment is a
directory, and every
ignored file that cannot be rebuilt in the examples above, `.env` and
`local.env`, is a single file. Requiring a trailing `/` on the `!!` line
keeps them out whatever the rest of the test says.

A `.git` at any depth makes an entry Unknown. In a scratch worktree, a
`.venv/` holding `src/pkg/.git/`, the shape `pip install -e git+…` leaves, printed one
`!! .venv/` line, and `git worktree remove` deleted the nested repository
with the rest. `find .venv -name .git -print -quit` printed
`.venv/src/pkg/.git`, and printed nothing on a `node_modules/` without one.
Checking only the top of the entry would have passed it.

The markers:

- The Cache Directory Tagging specification defines `CACHEDIR.TAG`, whose
  first 43 bytes are `Signature: 8a477f597d28d172789f06886806bc55`, as marking
  its directory as regenerable cache. It says to treat the tag as a hint and
  to check that it is a regular file with the exact header
  (<https://bford.info/cachedir/>). pytest writes one into `.pytest_cache`
  (`_pytest/cacheprovider.py`); `.ruff_cache` and `.mypy_cache` directories
  in public repositories hold one, and Cargo writes one into a `target/` it
  creates itself. The
  skill accepts the hint only on a worktree whose work is already proven to
  have landed.
- Every Python virtual environment holds a `pyvenv.cfg` with a `home` key
  (<https://docs.python.org/3/library/venv.html>). The environment is rebuilt
  from the project's own requirements.
- A `.gitignore` containing `*` is not accepted. venv writes one on Python
  3.13 and later, and pytest writes one, but a person who wants a folder kept
  out of git without editing the repository's `.gitignore` writes the same
  file. In a scratch worktree, a `notes/` holding that `.gitignore` and a
  `todo.txt` printed a single `!! notes/` line, indistinguishable from a
  cache's. The venv case is covered by `pyvenv.cfg` anyway.

The table needs a tracked project file in the same directory as the entry,
which is how kondo decides a directory is a project's artifact: a marker file
plus a list of artifact directories per ecosystem
(<https://github.com/tbillington/kondo/blob/master/kondo-lib/src/lib.rs>). The
rows are kondo's, kept to ecosystems whose marker is a file the project
tracks. Kondo's Unity marker, `Assembly-CSharp.csproj`, is one Unity
generates and projects ignore, so a tracked-file test would never meet it, and
the row is left out. The name alone is not enough: kondo's own list has
`build`, `vendor` and `bin` under several ecosystems, and a `build/` with no
`build.gradle` beside it may hold anything. Godot's documentation calls
`.godot/` "project cache data" to ignore "to avoid bloating your version
control repository with generated data"
(<https://docs.godotengine.org/en/stable/tutorials/best_practices/version_control_systems.html>).
GitHub's Godot template lists `.import/` with `.godot/`, under Godot-specific
ignores (<https://github.com/github/gitignore/blob/main/Godot.gitignore>).

In a scratch repository tracking `game/project.godot` and `src/app.py`,
`git ls-files -- ':(glob)game/project.godot'` printed it, and
`':(glob)src/*.py'` printed `src/app.py` but not `src/deep/x.py`, where the
plain pathspec `src/*.py` printed both. Without `:(glob)`, a `*` in a
pathspec matches across `/`, so a `.py` file anywhere below would vouch for a
directory it never generated.

This is a change from the first version of the skill, which sent every
landed worktree holding ignored files to the question. The reason it gave, no
command tells a cache from a `.env`, still holds for git; the test above is
the evidence that was missing.

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

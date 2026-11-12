---
name: clean
description: >-
  Cleans up a checkout once work is finished, usually after its pull request
  merged. Fetches, returns the checkout to the default branch and fast-forwards
  it, then deletes what the finished work left behind and only that: local
  branches whose work is proven to have landed (a squash merge counts), linked
  worktrees that are clean and whose branch landed, the state files other ez
  skills keep in the git directory, and scratch files this session made.
  Anything it cannot prove is finished stays and is listed with the reason:
  unmerged branches, dirty worktrees, tracked changes, stashes. A landed
  worktree that holds ignored files is removed only on the user's word.
  Only reads a remote, never writes to it, never forces, never discards
  uncommitted work. Works in any git repository and any language. Use this
  when work is done and the checkout should go back to a clean default branch,
  in preference to running `git branch -D`, `git worktree remove`, or
  `git clean` directly — invoked by the user via /ez:clean, or by the model
  when the user says the PR merged or the work is finished, or asks to clean
  up.
argument-hint: "[manual]"
allowed-tools: Bash, Read, Grep, Glob
---

# ez:clean

Put the checkout back the way it was before the work started: on the default
branch, level with the remote, with no branch, worktree, or file left over from
work that has landed. The chain's other skills each leave something behind, and
this one removes it.

Every deletion here needs **evidence that the work it holds is somewhere
else**. The skill does not judge what looks old or unused. A thing with no
evidence stays, and the report says why, so the user can remove it by hand in
one command. Leaving a finished branch behind costs a line in `git branch`;
deleting an unfinished one costs the work.

**A project skill wins.** If this repository carries its own cleanup skill —
`.claude/skills/*clean*/SKILL.md` — that skill knows the house conventions and
this one does not. Name it and stop rather than running.

## Hard rules — never violate

- Of the remote, run only `git fetch --prune`,
  `git ls-remote --symref <remote> HEAD`, and read-only `gh` calls. NEVER
  push, never delete a remote branch, never merge or close a PR.
- NEVER discard uncommitted work: no `git restore`, `git checkout -- <path>`,
  `git reset --hard`, `git stash` in any form, and no `--force` on
  `git switch` or `git worktree remove`. Tracked changes stay exactly where
  they are.
- NEVER run `git clean` without naming each path, and never with `-x` or `-X`.
  Ignored files are not this skill's: they hold build caches that cost time to
  rebuild and `.env` files that cannot be rebuilt, so a worktree holding them
  is removed only on the user's word.
- Delete a branch only on one of the four proofs in **When a branch has
  landed**. An upstream marked `[gone]` is not one, since closing a PR unmerged
  deletes the remote branch too. Neither is age, a name, or the user having
  moved on.
- NEVER delete the default branch, a protected branch, or the branch checked
  out in a worktree that is staying.
- NEVER move or reset a branch to make it match the remote. A default branch
  that will not fast-forward is reported and left.
- Of `gh stack`, run only `view` and `unstack --local`. `sync` and `rebase`
  rewrite history and force-push; they are the user's.
- An untracked file this session did not create is the user's until they say
  otherwise. Deleting one has no undo, so that is the one question this skill
  stops for. A `picked` plan that may have been built rides in the same
  question, and so do a state file that may describe those files and a landed
  worktree that holds ignored files, which `git worktree remove` would delete
  with it.
- Print every branch you delete with its tip SHA. That line is the undo:
  `git branch <name> <sha>` restores it until git's garbage collection runs.

## Inputs

`/ez:clean [manual]`. With `manual`, or when the user asks to see the plan
first, print the Phase 3 plan and stop for approval before Phase 4. Stay in
that mode for the rest of the session.

Invoke this skill yourself when the user says the work is finished or the PR
merged, or asks to clean up. Opening a PR is not that: the branch still has a
review ahead of it. Neither is a review that came back clean.

## When a branch has landed

Test each local branch against `<remote>/<default>` as it stands after the
fetch. The first proof that holds settles it. Record which one, since the
report names it.

1. **Ancestor.** `git merge-base --is-ancestor <branch> <remote>/<default>`
   succeeds. The branch was merged with a merge commit or fast-forwarded, or
   never had a commit of its own; either way the name is all that goes.
2. **Merged PR at this tip.** With `gh` authenticated,
   `gh pr list --head <branch> --state merged --json number,headRefOid,baseRefName`
   returns a PR based on the default branch whose `headRefOid` is the branch's
   tip, or has the tip as an ancestor
   (`git merge-base --is-ancestor <tip> <headRefOid>`, when that commit is in
   the local object store). Branch names get reused, which is why the name
   alone never counts. A tip with commits past `headRefOid` holds work the PR
   never carried: not landed.
3. **Nothing left to merge.** Merging the branch into the default branch would
   change nothing:

   ```sh
   git merge-tree --write-tree <remote>/<default> <branch>
   git rev-parse "<remote>/<default>^{tree}"
   ```

   The two print the same tree. This catches a squash merge or a rebase merge
   with no PR to ask, and work squash-pushed straight to the default branch.
   A conflict (exit 1) or a different tree fails this proof.
   `merge-tree --write-tree` needs git 2.38; on an older git, skip this proof.
4. **Its diff landed as one commit.** The branch's whole change matches a
   commit the default branch gained after the branch left it. With `<base>`
   from `git merge-base <remote>/<default> <branch>`:

   ```sh
   git diff-tree -p <base> <branch> | git patch-id --verbatim
   git rev-list <base>..<remote>/<default> | git diff-tree --stdin -p | git patch-id --verbatim
   ```

   The first ID the first command prints is among the first column of the
   second. This catches a squash that a later commit on the default branch
   edited over, where proof 3 meets a conflict. `--verbatim` keeps whitespace
   in the hash, so a branch whose unlanded commit only changes whitespace does
   not match. When either command prints nothing, as on a git that rejects
   `--verbatim`, this proof fails.

A branch none of these holds for is kept. Give its reason in a clause: an open
PR and its number, a PR closed unmerged, commits ahead with no PR, commits past
the merged PR's head.

## Workflow

### Phase 0 — Preflight
Run `git rev-parse --absolute-git-dir --show-toplevel` in the checkout the
skill was invoked in and record the two lines as `<git-dir>` and `<checkout>`,
quoted in shell commands. Outside a git repository, say so and stop. The
checkout is a linked worktree when
`git rev-parse --path-format=absolute --git-common-dir`, recorded as
`<git-common-dir>`, names a different directory from `<git-dir>`.

1. An operation in flight ends the run: `<git-dir>` holds `rebase-merge`,
   `rebase-apply`, `MERGE_HEAD`, `CHERRY_PICK_HEAD`, `REVERT_HEAD`, or
   `BISECT_LOG`. Name it and stop. Finishing or aborting it is the user's.
2. `git status --porcelain=v1 --untracked-files=normal`,
   `git branch --show-current`, and `git stash list`. Split the status into
   tracked changes and untracked files. The flag overrides
   `status.showUntrackedFiles=no`, which would hide every untracked file.
3. Find the default branch and its remote. The remote is `origin`, or the only
   one there is. The default branch is the name after `<remote>/` in
   `git symbolic-ref --short refs/remotes/<remote>/HEAD` (`origin/main` gives
   `main`); when that is unset, the name after `refs/heads/` on the `ref:` line
   of `git ls-remote --symref <remote> HEAD`. If that fails or prints no
   `ref:` line, stop and say so, as step 5 does when the fetch fails. With no
   remote at all, the default is the local `main` or `master`, the proofs run
   against it, and the fetch and the fast-forward are skipped; say so.
4. Protected branches are the default branch, `commit.protectedBranches` from
   `.claude/ez.json`, and `main` and `master`.
5. `git fetch --prune <remote>`. If it fails, stop: without it every proof
   would be read against a stale default branch.
6. `gh auth status`. Without it the PR proof is skipped and the others
   still run; say so in the report.

### Phase 1 — Classify
Read-only. Nothing is deleted before Phase 4.

**Branches.** List them with
`git for-each-ref --format='%(refname:short) %(objectname:short) %(upstream:short) %(upstream:track)' refs/heads`
and run the proofs on every one that is not protected. Each is `landed` with
its proof, or `kept` with its reason.

**The stack.** When `gh stack view --json` tracks the current branch, read its
entries. If every entry has landed, the stack is finished: Phase 4 drops its
local tracking once the run is on the default branch, by the number on the
`Stack #<n>` line of `gh stack view --short`. With no such line it cannot be
dropped from another branch. If some entries have not landed, the stack is
still live. In a live stack, and in a finished one whose record Phase 4 did
not drop, a landed entry is deleted only when the view marks it `isMerged`,
because `gh stack` tolerates a missing branch only once it has recorded the
merge. Any other landed entry is kept, and the report names
`gh stack sync --prune` as the user's next step: it restacks what is left and
removes merged branches itself.

**Other stacks.** `view` shows only the current branch's stack, so read the
others from the JSON files `gh stack` keeps, one per worktree:
`<git-common-dir>/gh-stack` and each `<git-common-dir>/worktrees/*/gh-stack`
that exists. A landed branch that a stack other than the current one names,
as its `trunk.branch` or in its `branches[].branch`, is deleted only when
every entry that names it has `"pullRequest": {"merged": true}`. Any other is
kept. When its entry has a PR, the report names `gh stack sync --prune`, which
records the merge and prunes it. An entry with no PR is never recorded as
merged, so the report says that stack still lists it.

**Worktrees.** `git worktree list --porcelain`. For each linked worktree other
than the one this session is in:

- `prunable` means its directory is already gone. Phase 4 clears the record.
- `locked` is kept. A lock is someone's claim that it is in use.
- Touched in the last hour is kept: the `HEAD` file in its own git directory
  (`git -C <path> rev-parse --absolute-git-dir`) was modified within the hour,
  so a session may be using it. A fresh worktree on a branch with no commits
  passes every other test here.
- Otherwise it is removable when
  `git -C <path> status --porcelain=v1 --ignored --untracked-files=normal`
  prints nothing and its work is elsewhere. Its work is elsewhere when its
  branch is `landed`, or its HEAD is detached at a commit
  some branch or the default branch contains
  (`git -C <checkout> branch -a --contains <sha>` prints at least one line).
  The throwaway worktrees `/ez:review`, `/ez:commit`, and `/ez:fix` make are
  detached at a branch's commit, so one that outlived a crashed run qualifies.
  A worktree under `.claude/worktrees/` gets the same test as any other.
- When its work is elsewhere and that status prints only `!!` lines, it goes
  to Phase 2 and is removable once the user approves it there.
  `git worktree remove` deletes ignored files without complaint, and no
  command tells a `node_modules/` that costs a rebuild from a `.env` that
  cannot be rebuilt. `--untracked-files=normal` overrides
  `status.showUntrackedFiles`, which at `no` hides every line and at `all`
  lists each file of an ignored directory on its own line.
- Anything else is kept: uncommitted or untracked files, a branch that has
  not landed, a detached commit nothing else holds, a worktree Phase 2 did
  not approve.

A branch checked out in a kept worktree is kept with it.

**State files.** The files the chain writes to `<git-dir>`, with the test for
each. "Its branch" is the branch its first line names. The tree is *clear*
when it has no tracked changes and every untracked file in it is Scratch,
which Phase 4 deletes (see **Untracked files**). With no tracked changes but
some Unknown files, nothing shows whether a state file describes them, so a
file whose test below asks for a clear tree goes to Phase 2 with them and is
stale once the user approves it there.

- `ez-commit-plan.md`, `ez-review.md` — stale when its branch is `landed` or
  no longer exists. When its branch is protected, it described a working tree
  that was not yet on a branch: stale when the tree is clear.
- `ez-plan.md` — its heading ends in a status. `built` is stale when its
  branch is `landed`, gone, or protected with a clear tree.
  `framed` may be a plan whose options are waiting for the user's pick, with
  the scouts' brief only in this file: keep it and name its problem in the
  report. `picked` may be a plan nobody has built yet, or one built with a
  criterion unmet or in another session, which `/ez:plan` leaves `picked`
  too. When its branch is `landed`, gone, or protected with no tracked
  changes, and
  `git log --format=%h $(git merge-base <sha> <remote>/<default>)..<remote>/<default> -- <paths>`
  lists commits, `<sha>` being the heading's and `<paths>` its `Touches:`
  line, it goes to Phase 2 with those commits and is stale once the user
  approves it there. Otherwise keep it and name its problem in the report.
- `ez.patch`, `ez-fix.patch` — staging scratch, stale when the tree has no
  tracked changes.

A removed worktree takes its own git directory, and any state files in it,
with it.

**Untracked files.** From `git status`, ignored files excluded. Sort each:

- **Scratch** — a file this session created as a means to an end and nothing
  tracked refers to: a debug script, a captured log, a throwaway fixture, a
  dump made to inspect something. You know which these are because you wrote
  them in this conversation. After a compaction or in a fresh session you do
  not, and nothing is scratch.
- **Unknown** — everything else, whatever its name or age suggests.

### Phase 2 — Ask about unknown files
Skip this phase when there are none and no `picked` plan or worktree was sent
here. Otherwise list what was sent, one line each:

- an unknown file with its size and modification date, and a directory with
  its file count;
- a `picked` plan with its problem and the commits that touched its files;
- any other state file with its heading and the paths it cites, or that it
  cites none;
- a worktree with its path and each entry its `!!` lines name, with that
  entry's size.

Scratch is not listed: Phase 4 deletes it either way. Ask with AskUserQuestion
what to do, `Keep them all` listed first, then `Delete them all`, then
`Let me pick`. On the last, take the paths the user names. Ask once; a file
the user does not name is kept.

### Phase 3 — Print the plan
Print what Phase 4 will do, in the report's shape below with the verbs in the
future. Then keep going in the same turn: every deletion in it is either
backed by a proof or was just approved in Phase 2. **With `manual`, STOP here
instead.**

### Phase 4 — Act
In this order, since each step frees the next. A step that fails is reported
and the rest go on, unless a later step depends on it.

1. **Leave the current branch.** With no tracked changes and outside a linked
   worktree, `git switch <default>`, whether or not the current branch landed;
   an unfinished branch is kept and is one `git switch` away. If git refuses
   because an untracked file would be overwritten, stay and report it.
   With tracked changes, stay put: the changes belong to this branch, and the
   branch is kept this run whatever the proofs said. In a linked worktree, stay
   as well, since the default branch is checked out elsewhere.
2. **Drop a finished stack's tracking** with `gh stack unstack --local <n>`,
   only when step 1 ended on the default branch, switched or already there,
   and Phase 1 found the stack's number. It removes the local record and
   leaves GitHub's alone. Otherwise, or when it fails, the record stays, and
   step 5 handles the stack as live.
3. **Bring the default branch level.** On it,
   `git merge --ff-only <remote>/<default>`. Ahead of the remote is fine and
   is reported with the count. Diverged, it fails; report both counts and
   leave it.
4. **Worktrees.** `git worktree remove <path>` for each removable one, with no
   `--force`; one that holds ignored files is removable only once Phase 2
   approved it. Git refuses a worktree that turned dirty since Phase 1; report
   it as kept. Then `git worktree prune`.
5. **Branches.** For each `landed` branch that no kept worktree holds:
   `git branch -D <branch>`. Git's `-d` check compares against the branch's
   upstream, or HEAD without one, never `<remote>/<default>`, so it can refuse
   a branch any of the proofs cleared. Run the proof again immediately
   before each `-D` if anything has moved since Phase 1. An entry of a stack
   whose record is still kept is deleted only when the view marked it
   `isMerged`; the rest are kept, as **The stack** says.
6. **State files.** `rm` each stale one by its full path under `<git-dir>`.
7. **Untracked files.** For scratch and for what Phase 2 approved,
   `git clean -f -- <path>…`, adding `-d` only for a directory that is on the
   list. `git clean` leaves a tracked path alone, which is the reason to use
   it over `rm`.

### Phase 5 — Report
Run `git status -sb`, `git branch`, and `git worktree list` and report from
what they print, not from what Phase 4 intended:

```markdown
## Clean — <repo> · on <branch> @ <short sha> (<level with | n ahead of | diverged from> <remote>/<default>)

Deleted branches:
- `feat/retry` (was a1b2c3d) — PR #41 merged
- `fix/typo` (was 9f8e7d6) — nothing left to merge
Removed worktrees: `.claude/worktrees/retry`; pruned 1 missing.
Removed files: `ez-review.md`, `ez-commit-plan.md` from the git directory; `debug.log`, `scratch/probe.py`.

Kept:
- `feat/cache` — PR #44 open
- `spike/queue` — 3 commits, no PR
- worktree `../exp` — 2 uncommitted files
- worktree `.claude/worktrees/probe` — touched within the hour
- `notes.txt` — untracked, kept at your word
- 2 stashes
- `ez-plan.md` — picked, not built: <its problem>
```

Leave out any line with nothing under it. When nothing needed doing, say so in
one line with the branch and SHA. Mention the checkout's own ignored files only
to say they were not looked at. When the run stayed off the default branch,
say why first, since that is the thing the user asked for. When invoked inside
a linked worktree
whose branch landed, close by saying it can be removed from the main checkout
with `git worktree remove <path>`, or by leaving the worktree session; this
skill does not remove the directory it is standing in.

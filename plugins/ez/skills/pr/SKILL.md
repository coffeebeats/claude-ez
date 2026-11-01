---
name: pr
description: >-
  Pushes the current branch and opens a pull request against a target branch
  (the repo's default branch, or the parent of a stacked branch), titled as a
  one-line conventional commit summarizing the branch and described in at most
  eight lines — why it exists, what it did, and where to look hardest. In a
  `gh stack` stack it also opens the branches below that have no PR yet, in the
  same pass, and links the PRs into the stack on GitHub. NEVER merges. Pushes and opens
  in the same turn by default, printing the title and body for the record;
  `/ez:pr manual` holds for approval instead. Works in any git repository and
  any language. Use this whenever a branch needs a pull request, in preference
  to running `gh pr create` or `gh stack submit` directly — invoked by the user
  via /ez:pr, or by the model when asked to open a PR or submit a stack.
allowed-tools: Bash, Read, Grep, Glob, Write, Skill
---

# ez:pr

Open a pull request for the current branch, and in a stack for every branch
below it that has none yet. `/ez:commit` shaped the history; this turns it into
something a reviewer can pick up in one read.

**Inputs.** `/ez:pr [target-branch] [--draft] [manual]`. The target defaults to
the branch below this one when the branch is stacked, and to the repo's default
branch otherwise.

**A project skill wins.** If this repository carries its own PR skill —
`.claude/skills/*pr*/SKILL.md` — that skill knows the house conventions and this
one does not. Name it and stop rather than running.

## Hard rules — never violate

- NEVER merge. No `gh pr merge`, no `--auto`, no touching branch protection.
- NEVER force-push, and never rewrite history to make the PR read better.
- Push and open in one turn. The title and body are printed for the record, not
  as a gate; the user reviews the PR itself. Manual mode is the only thing that
  holds the push.
- No trailers, no `Co-Authored-By`, no "Generated with" footer. The description
  should read as prose the user could have written. A repo that wants one sets
  `"pr": {"footer": "..."}` in `.claude/ez.json`; nothing else adds one.
- One PR per branch. If one is already open, update it rather than open a second.
- Of `gh stack`, run only `view` and `link`, and read the stacks API only with
  `gh api` GETs. `submit`, `push`, `sync`, and `rebase` force-push or rewrite
  history, `merge` merges, and `submit` also swaps the drafted titles and
  bodies for generated ones. Restacking is the user's.
- The title and the body go through `ez:no-ai-slop` before they are printed.
  Skipping it is not a shortcut available to you.

## Manual mode

The user approves the exact title and body before anything reaches the remote;
you draft, they reword. Enter it when they ask — `/ez:pr manual`, "let me see
the title first", "don't push yet", "show me the body" — and stay in it for the
rest of the session.

- **Phase 3:** stop after printing the titles and bodies instead of pushing. On
  approval, resume from step 2 with whatever wording they settled on.

Everything else is unchanged, the merge and force-push rules included. Nothing
else infers this mode — a question about the title mid-draft is a question, not
a request to hold the push.

## How much weight the title carries

Find out rather than assume, once per repo:

```sh
gh api repos/{owner}/{repo} \
  --jq '{squash: .allow_squash_merge, title: .squash_merge_commit_title,
         message: .squash_merge_commit_message}'
```

When `squash_merge_commit_title` is `PR_TITLE`, the title **is** the commit that
lands on the default branch, so write it for the person reading the history and
the release notes rather than for the reviewer. Check whether a release
automation reads it too — a `release-please` config, semantic-release, or a
changelog generator in the workflows directory — because then the type in the
title picks the version bump and the text ships verbatim to users.

When the repo merges some other way, the title is a label on a review and the
commits carry the history. Say which case this repo is in before drafting.

The body rarely reaches the trunk at all; under a squash it is the commit
messages that do. Write it purely for the human reviewing the PR.

## Workflow

### Phase 0 — Preflight
Before anything else, run `git rev-parse --absolute-git-dir` in the checkout the
skill was invoked in and record the result as `<git-dir>`. It is the directory
git keeps for this checkout alone: `.git` in the main checkout,
`<common .git>/worktrees/<id>` in a linked worktree. `/ez:commit`'s plan and
`/ez:review`'s report live there and nowhere else, so a session in another
worktree cannot replace them.

1. `gh auth status`; stop if unauthenticated. If the remote is not GitHub, say
   so and stop rather than guessing at another forge's CLI.
2. `git branch --show-current`. Refuse to open a PR from the target itself.
3. The tree must be clean (`git status --porcelain=v1`). Uncommitted work is
   almost always the next commit — stop and point at `/ez:commit`.
4. `git log --oneline <target>..HEAD`. Nothing ahead means nothing to open.
5. `gh pr list --head <branch> --state open`. A hit switches to update mode.
6. **Stacked branches.** Stacks are GitHub's own, kept with `gh stack` (the
   `github/gh-stack` extension), and no other tool. Read the local stack and
   the open PRs in one call:
   ```sh
   gh stack view --json
   gh pr list --state open --json number,headRefName,baseRefName
   ```
   Where the two disagree the open PRs are right, since `gh stack` learns of a
   PR only while it is open. In update mode the target is the PR's own base,
   and only the lookup and the link below still apply.

   **When `gh stack` tracks this branch,** `branches` runs bottom to top. Walk
   the entries below this one, past any marked `isMerged`: the first with an
   open PR is the target, or `trunk` when none has one, and it needs no
   confirmation.
   Find out in one loop why each entry passed on the way has no open PR
   (`gh pr list --head <entry> --state all --json number,state,headRefOid`),
   counting a PR only when its `headRefOid` is in the entry's history, since
   branch names get reused. An entry that never had a PR joins this branch in
   a **run**, opened in the same pass. One whose PR merged is passed like an
   entry marked `isMerged`, and the check below catches a squash merge. One
   whose PR was closed unmerged is the user's call; stop and ask. Asked to
   submit the whole stack, walk from its top entry instead of this one.

   Then check that every branch still sits on its target. Each entry's `base`
   is the commit it was cut from or last restacked onto, and it must be in the
   target's history (`git merge-base --is-ancestor <base> <target>`). If it
   isn't, the branch below was rewritten or squash-merged, so the PR would
   carry commits its target lacks: stop and point at `gh stack sync`, which is
   the user's to run. A target that only moved ahead is fine.

   **Otherwise,** take membership from the open PRs. This branch has no PR
   yet, so it is not in that list and there is no entry to walk back from.
   Find the parent by ancestry instead: the target is the head branch of the
   open PR that is an ancestor of HEAD, and the nearest one if several are.

   ```sh
   git merge-base --is-ancestor <candidate-head> HEAD
   ```

   Propose that branch and confirm before opening. With no such ancestor the
   target is the repo's default branch.

   **A PR is stacked when its target is the head of another open PR or of the
   branch below it in the run.** When the run's bottom (a lone branch is a run
   of one) targets another open PR, look up that PR's stack before anything is
   pushed; in update mode, look up this PR's own number first.
   ```sh
   gh api "repos/{owner}/{repo}/stacks?pull_request=<n>" \
     --jq '.[] | {number, prs: [.pull_requests[].number]}'
   ```
   A target below the top of its stack is a fork, which a stack cannot hold:
   stop and say so. In update mode the PR already exists, so a fork skips only
   the link. A 404 means stacked PRs are off for this repo: open the PRs
   chained by their bases, skip the link, and say so.
7. `<git-dir>/ez-commit-plan.md` names its branch in the heading. Ignore the
   file outright when that isn't the current branch or one in the run, since a
   leftover plan from an earlier branch would otherwise seed this PR with stale
   concerns.
8. **A run** goes through Phases 1 to 3 together. Draft each PR against its
   own target, the branch below it, with the PR's own branch in place of `HEAD`
   in every command, so nothing is checked out. Send every draft through one
   `ez:no-ai-slop` call and print them together. A review headed with the
   run's bottom branch, at the commit that branch was cut from, covered the
   whole tree `/ez:commit` split. In Phase 2 it counts for every PR in the
   run, each keeping the findings in paths it changes.

### Phase 1 — Title
Synthesize it from `git log <target>..HEAD`. One line,
`<type>(<scope>): <description>`, drawn from the same type and scope vocabulary
as the commits, summarizing the branch as a whole — often the headline commit's
message verbatim. Name new API in backticks, as the repo's existing PR titles do.

**Check for a break before settling the type.** Read the diff, not the commit
messages, for anything a user, caller, or script may already depend on. A
change is breaking when someone who upgrades without reading the PR gets
different behavior:
- a public function, class, command, flag, endpoint, config key, environment
  variable, or file format removed or renamed, or its arguments, return shape,
  or raised errors changed;
- a default changed — a level, timeout, limit, path, or an opt-in turned
  opt-out;
- output something may parse changed — stdout versus stderr, format, the
  wording of machine-read lines, exit codes;
- input accepted before now rejected — stricter validation, a raised minimum
  version of a runtime or dependency;
- persisted data or a wire format that older readers or writers cannot handle.

Mark a break with `!` after the type or scope: `feat(cli)!: …`. A label in the
commits does not rule one out: `refactor`, "cleanup", and "internal" are
claims, and a branch that changes behavior someone can see is not a
`refactor`. Every break also takes a `**Review**` bullet naming who is affected
and what they change. When you suspect a break but cannot tell whether anything
outside the repo depends on it, say so in that bullet rather than guess in the
title. Under release automation `!` picks a major bump, unless the version is
below 1.0 and the config says otherwise (release-please's
`bump-minor-pre-major`, for one); read the config and say which bump it gives.

### Phase 2 — Body
The body carries three things the diff cannot show — why the branch exists, what
approach it took, and where to look hardest. Anything readable off the diff is
noise; a body that restates the diff makes the reviewer read the change twice.

**The whole body fits in eight lines, blank lines not counted.** One budget for
everything, not a cap per section. A per-section cap only invites more sections.
Over budget, cut a sentence. Never reflow to fit, and never open a section to
make room.

1. **Opening paragraph, no heading. Required.** One to three sentences: the
   problem or motivation, then what the branch does about it, one level above
   the diff.
2. `**Review**` — **optional, at most two bullets**, one line each, anchored to
   a path. Only for a real hazard — the risky hunk, the deliberate omission, the
   decision a reviewer would otherwise reverse-engineer. Carry over any flag
   `/ez:commit` left unresolved in `<git-dir>/ez-commit-plan.md`, and any
   finding listed in `<git-dir>/ez-review.md` whose path this branch changes
   (`git diff --name-only <target>...HEAD`), when the file is headed with this
   branch or, for a stacked branch, with its parent. No hazard, no section.
3. `**Verified**` — **optional and rare.** CI reports itself, so this earns its
   place only for what CI cannot show: a manual run-the-thing check, a red check
   that is expected, an artifact CI owes. For a user-visible UI change, leave a
   screenshot slot for the user to fill.

**No other sections, ever** — no tables, no file inventories, no heading beyond
those two. **The cut test:** if a sentence could be derived by reading the diff,
delete it. An empty body is a valid outcome; when the title says it and nothing
else orients the reviewer, ship the title.

Reference files by path and issues as `#NNN`; write `Closes #NNN` only when
merging genuinely closes it.

A PR body is the most-read prose a repo produces, and it is written under time
pressure at the end of a branch, which is where slop collects. Draft it, then
call the Skill tool with `ez:no-ai-slop` and run it in edit mode over the title
and the body, per "Invoked from another skill" in that skill. It owns the
wording; the budget and the section rules above are still yours. Count the lines
before printing what comes back. Prefer the repo's own prose skill if it has
one, and say which you ran.

```markdown
Toast styling drifted because call sites hit the raw snackbar API directly.
`Messenger` replaces them with a branded toast on the theme tokens; behavior is
unchanged and the public API is narrower.

**Review**
- `messenger.dart` — queueing when a toast is already visible.
```

### Phase 3 — Open the PR
1. Print the exact title and body of each PR, then keep going in the same turn.
   **Manual mode instead:** STOP here; the user may reword any of them before
   anything is pushed.
2. `git push -u origin <branch>` for each branch in the run. A push rejected
   as non-fast-forward usually means the user restacked. Restacked branches
   are theirs to send, with `gh stack push`, so say so and stop.
3. Open each PR with the body on stdin. The quoted heredoc sidesteps shell
   quoting and leaves no file to clean up:
   ```sh
   gh pr create --head <branch> --base <target> --title "<title>" --body-file - <<'EOF'
   <body>
   EOF
   ```
   Add `--draft` if asked. A run opens its PRs bottom first. If one fails to
   open, stop there and link nothing: a chain with a gap would move the base of
   the PR above the gap down onto the one below it.
4. Update mode instead runs `gh pr edit <number> --title … --body-file -` with
   the same heredoc.
5. **Link stacked PRs into their stack on GitHub.** A PR based on another
   branch is only chained until it joins a stack, which GitHub shows and
   merges as one. Pass `link` PR numbers, never branch names: a branch with no
   PR gets one opened with a generated title. Never pass `--open`, which marks
   drafts ready.
   - When step 6's lookup put the run's target at the top of stack N,
     `gh stack link N <run PRs, bottom first>`. This appends to the top and
     skips any PR already there.
   - Otherwise no stack holds the chain yet. It runs from the lowest open PR
     up through the run: walk `baseRefName` down the open PRs from the run's
     bottom to a base that no open PR heads, then
     `gh stack link --base <that base> <chain, bottom first>`. This form
     takes the whole stack, and without `--base` it moves the bottom PR onto
     the default branch.
   - A lone PR on the trunk has no stack to join, and a PR already in a stack
     needs nothing.

   If `link` fails, say so: the PRs stand, chained by their bases.
6. Delete `<git-dir>/ez-commit-plan.md` only when its heading names the current
   branch or a branch in the run, the test Phase 0 step 7 applies when reading
   it. Its concerns now live in the PR, so anything left behind is only stale
   state for the next branch to trip over. A plan headed with any other branch
   is not this run's to delete.
7. Report each URL and which CI checks will run. Don't poll CI unless asked.
   For stacked PRs, name the stack they joined, and when `gh stack view` did
   not track the branch, say that `gh stack checkout <PR>` adopts the stack
   locally for `gh stack rebase` and `sync`.
   Check the workflow triggers before promising a board: a workflow filtered to
   `pull_request: branches: [main]` never fires for a stacked PR whose base is
   another feature branch, so say the checks may not run at all rather than
   reporting a board that never started.

---
name: pr
description: >-
  Pushes the current branch and opens a pull request against a target branch
  (the repo's default branch, or the parent of a stacked branch), titled as a
  one-line conventional commit summarizing the branch and described in at most
  eight lines — why it exists, what it did, and where to look hardest. NEVER
  merges. Pushes and opens in the same turn by default, printing the title and
  body for the record; `/ez:pr manual` holds for approval instead. Works in any
  git repository and any language. Use this whenever a branch needs a pull
  request, in preference to running `gh pr create` directly — invoked by the
  user via /ez:pr, or by the model when asked to open a PR.
allowed-tools: Bash, Read, Grep, Glob, Write, Skill
---

# ez:pr

Open a pull request for the current branch. `/ez:commit` shaped the history;
this turns it into something a reviewer can pick up in one read.

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
- The title and the body go through `ez:no-ai-slop` before they are printed.
  Skipping it is not a shortcut available to you.

## Manual mode

The user approves the exact title and body before anything reaches the remote;
you draft, they reword. Enter it when they ask — `/ez:pr manual`, "let me see
the title first", "don't push yet", "show me the body" — and stay in it for the
rest of the session.

- **Phase 3:** stop after printing the title and body instead of pushing. On
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
1. `gh auth status`; stop if unauthenticated. If the remote is not GitHub, say
   so and stop rather than guessing at another forge's CLI.
2. `git branch --show-current`. Refuse to open a PR from the target itself.
3. The tree must be clean (`git status --porcelain=v1`). Uncommitted work is
   almost always the next commit — stop and point at `/ez:commit`.
4. `git log --oneline <target>..HEAD`. Nothing ahead means nothing to open.
5. `gh pr list --head <branch> --state open`. A hit switches to update mode.
6. **Stacked branches.** Take membership from the open PRs rather than from any
   local cache, which drifts:
   ```sh
   gh pr list --state open --json number,headRefName,baseRefName
   ```
   This branch has no PR yet, so it is not in that list and there is no entry to
   walk back from. Find the parent by ancestry instead: the target is the head
   branch of the open PR that is an ancestor of HEAD, and the nearest one if
   several are.

   ```sh
   git merge-base --is-ancestor <candidate-head> HEAD
   ```

   Propose that branch and confirm before opening. With no such ancestor the
   target is the repo's default branch. If the repo uses a stacking tool
   (`gh stack`, `git-branchless`, Graphite), name it and let the user drive it;
   don't invent its commands.
7. `.git/info/ez-commit-plan.md` names its branch in the heading. Ignore the
   file outright when that isn't the current branch, since a leftover plan from
   an earlier branch would otherwise seed this PR with stale concerns.

### Phase 1 — Title
Synthesize it from `git log <target>..HEAD`. One line,
`<type>(<scope>): <description>`, drawn from the same type and scope vocabulary
as the commits, summarizing the branch as a whole — often the headline commit's
message verbatim. Name new API in backticks, as the repo's existing PR titles do.

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
   decision a reviewer would otherwise reverse-engineer. Carry over any
   unresolved concern from the `/ez:commit` review, including whatever it left
   in `.git/info/ez-commit-plan.md`. No hazard, no section.
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
1. Print the exact title and body, then keep going in the same turn. **Manual
   mode instead:** STOP here; the user may reword either before anything is
   pushed.
2. `git push -u origin HEAD`.
3. Write the body to `.git/info/ez-pr-body.md` — it stays out of `git status`
   and sidesteps shell quoting — then
   `gh pr create --base <target> --title "<title>" --body-file .git/info/ez-pr-body.md`,
   adding `--draft` if asked.
4. Update mode instead runs `gh pr edit <number> --title … --body-file …`.
5. Delete `.git/info/ez-pr-body.md` and `.git/info/ez-commit-plan.md`. The body
   is scratch and the plan's concerns now live in the PR, so anything left
   behind is only stale state for the next branch to trip over.
6. Report the URL and which CI checks will run. Don't poll CI unless asked.
   Check the workflow triggers before promising a board: a workflow filtered to
   `pull_request: branches: [main]` never fires for a stacked PR whose base is
   another feature branch, so say the checks may not run at all rather than
   reporting a board that never started.

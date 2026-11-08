---
name: publish
description: >-
  Finishes a branch once review and fixes are done. Runs `/ez:comments` over
  every source file the branch changes and `/ez:no-ai-slop` over the prose it
  adds to docs, then lands the result with `/ez:commit` and pushes it with
  `/ez:pr`, which opens the pull request or updates the one already open. Each
  step keeps its own rules and its own stops. Works in any git repository and
  any language. Invoked by the user via /ez:publish.
argument-hint: "[target-branch] [--draft] [manual]"
disable-model-invocation: true
allowed-tools: Bash, Read, Grep, Glob, Edit, Skill
---

# ez:publish

Finish the chain in one go: tidy the comments and the prose across the whole
branch, commit what that and any earlier fixes left in the
tree, then push and open or update the PR. It is shorthand for
`/ez:comments`, `/ez:no-ai-slop`, `/ez:commit`, and `/ez:pr` in that order, and
adds no judgment of its own beyond deciding what each one is handed.

**A project skill wins.** If this repository carries its own publish skill,
`.claude/skills/*publish*/SKILL.md`, name it and stop rather than running.
When a step below defers to a project skill of its own, run that project skill
in its place and carry on with the next step.

## Hard rules — never violate

- Each step runs as its own skill, through the Skill tool, with its own hard
  rules intact. NEVER merge, never force-push, and never commit or push except
  through `/ez:commit` and `/ez:pr`.
- Keep every stop a step makes. `/ez:commit` waits for approval of a stack split
  and stops for its judgment calls; `/ez:pr manual` holds the push. Wait there,
  then resume this workflow at the step after the one that stopped. Being a
  shorthand is not permission to answer a step's question for the user.
- A step that fails without handing the user a choice ends the run: a preflight
  refusal, a rejected push. Report where it stopped and what is left, and never
  jump ahead to the push. A failing gate in `/ez:commit` is one of its stops:
  wait for the user's ruling, and resume only once the tree is clean.
- The polish passes reach only files the branch changes, never the rest of the
  repo, and never change behavior. Within those files the comment pass covers
  every comment and the prose pass only what the branch wrote.

## Inputs

`/ez:publish [target-branch] [--draft] [manual]`. Every argument is passed to
`/ez:pr` unchanged; `manual` holds the push for approval of the title and body,
and leaves `/ez:commit` committing as usual. A target given here also sets the
base in step 2.

## Workflow

### 1. Preflight

1. `gh auth status`, and stop if it fails. The run exists to publish, so it
   refuses before editing anything rather than after the commits land.
2. `git branch --show-current`, and `git status --porcelain=v1` with untracked
   files included.

### 2. Find the branch's change

The base is the first of these that applies:

- the target given as an argument;
- the base of the branch's open PR,
  `gh pr list --head <branch> --state open --json baseRefName`;
- the branch below this one in `gh stack view --json`, when the stack tracks it;
- the head of the nearest open PR that is an ancestor of HEAD, for a branch cut
  from another with `git switch -c`: list the heads with
  `gh pr list --state open --json headRefName` and test each with
  `git merge-base --is-ancestor <head> HEAD`;
- the repo's default branch, `gh repo view --json defaultBranchRef`.

When the current branch is itself the base or a protected branch, the change is
the working tree against `HEAD`, which `/ez:commit` will move onto a new branch,
and `<fork-point>` is `HEAD`; skip the `merge-base` command below.
Otherwise the change is everything since the fork point, committed or not. The
fork point is the newest commit `HEAD` shares with any copy of the base, local
or on a remote, since `git pull --rebase origin <base>` moves the branch and
leaves the local `<base>` behind. Given several refs, `git merge-base` answers
for all of them at once, so a copy that lags never pulls the fork point back.
A stacked parent's remote copy lags that way once the parent is rewritten and
not yet pushed. Call the first command's output `<fork-point>`:

```sh
git merge-base HEAD $(git for-each-ref --format='%(refname)' \
  "refs/heads/<base>" "refs/remotes/*/<base>")
git diff --name-only --diff-filter=d <fork-point>
git ls-files --others --exclude-standard
```

In a stack this is the checked-out branch's own diff. Branches below it keep
whatever they were published with; `/ez:pr` may still open their PRs in the
same pass.

Nothing changed and nothing ahead of the base means nothing to publish: say so
and stop. Otherwise sort the files:

- **Source** is every file in a language with comments, tests included. It goes
  to step 3.
- **Prose** is Markdown, reStructuredText, and plain-text docs: READMEs,
  `CLAUDE.md`, `AGENTS.md`, `.claude/**.md`, changelogs a person writes. It goes
  to step 4.
- **Skip** generated output, lockfiles, binaries, fixtures, and anything
  vendored from elsewhere, such as a directory carrying its upstream's licence.

Print the base and the two lists in a few lines before editing.

### 3. Comments

Call the Skill tool with `ez:comments` and pass it the source list. It audits
every comment in each file, old ones included, so a file the branch touches
comes out wholly in the house style. It runs its own `ez:no-ai-slop` detect
pass over the comment prose, prints its audit, and edits. Its closing pointer
at `/ez:commit` is this workflow's step 5. With no source files, skip this step
and say so.

### 4. Prose

Call the Skill tool with `ez:no-ai-slop` and run it in edit mode over the prose
list, per "Invoked from another skill" in that skill. Edit what the branch wrote
and leave the rest: a new file whole, and in a changed file only the passages
`git diff <fork-point> -- <file>` shows as added or changed.
Apply the edits with Edit. With no prose files, skip this step and say so.

### 5. Commit

When `git status --porcelain=v1` is clean, nothing was edited and no fix was
waiting, so skip to step 6 and say so. Otherwise call the Skill tool with
`ez:commit`. The tree holds the polish from steps 3 and 4 and anything
`/ez:fix` or the user left uncommitted, and `/ez:commit` slices all of it. When
it finishes, the tree must be clean; if the user held anything back, stop here,
since `/ez:pr` refuses a dirty tree.

### 6. Pull request

Call the Skill tool with `ez:pr` and pass this skill's arguments through. It
opens the PR, or updates the open one's title and body to match the new
commits, and reports the URL.

### 7. Report

Close with one line per step: what the comment pass deleted and trimmed, what
the prose pass changed, the commits that landed, and the PR's URL. Leave out
anything `/ez:pr` already printed.

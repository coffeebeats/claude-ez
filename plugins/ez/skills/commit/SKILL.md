---
name: commit
description: >-
  Takes a finished working tree and lands it as one or more branches, each
  split into an ordered sequence of one-line conventional commits, committed one
  slice at a time. Stages, proposes the message, and commits it; pass `manual`
  (or ask to commit yourself) to have it stage and hand off instead. Works in
  any git repository and any language. Use this whenever a finished tree needs
  committing, in preference to running `git commit` directly — invoked by the
  user via /ez:commit, or by the model when asked to commit.
allowed-tools: Bash, Read, Grep, Glob, Write, Edit
---

# ez:commit

Help the user land a batch of already-implemented working-tree changes, broken
into a digestible, ordered sequence of atomic commits across one or more
branches. The user comes to understand the work one commit at a time. You
stage each slice and commit it; **Manual mode** hands that last step back.

**A project skill wins.** If this repository carries its own commit skill —
`.claude/skills/*commit*/SKILL.md` — that skill knows the house conventions and
this one does not. Name it and stop rather than running.

## Hard rules — never violate

- NEVER run `git commit --amend`, `git merge`, `git rebase`, `git push`,
  `git pull`, or anything else that rewrites history or touches a remote. Plain
  `git commit` is the one write you own, and Manual mode takes even that away.
- NEVER stage or commit directly on a protected branch. Phase 0 creates and
  switches to a working branch before anything else happens; that default name
  needs no sign-off. Only a multi-branch **stack** split (Phase 2) needs its
  names approved before touching git.
- One slice per commit, in the planned order. Never batch several planned
  commits into one `git commit`, and never reorder them to save a step.
- Commit messages: single-line Conventional Commits. No body, no footers, no
  trailers, no `Co-Authored-By`.
- You are sorting existing changes, not rewriting them. Don't edit the user's
  code to make commits cleaner unless they ask.
- Don't rewrite comments while staging. Report what a comment linter finds and
  let the user run `/ez:comments` first, as its own commit.
- Holding the commit is not permission to skip a judgment call. STOP for
  anything genuinely ambiguous — a hunk that could belong to two commits, a
  Phase 1 concern the user should rule on, a gate failure that may want its own
  fix commit, a change to a file the user authored rather than you.

## Manual mode

The user reads each diff and runs `git commit` themselves; you only ever touch
the index. Enter it when they ask — `/ez:commit manual`, "just stage them",
"I'll commit", "let me review each one" — and stay in it for the rest of the
session. Phase 4 walks through each slice's key hunks before the message and
stops after presenting it instead of committing; everything else is unchanged.

## Repository configuration

Read `.claude/ez.json` at the repo root first. Every key is optional, and the
detection rules below fill in whatever it omits.

```json
{
  "gate": {
    "fast": ["cargo fmt --check", "cargo clippy -- -D warnings"],
    "full": ["cargo test --all"],
    "codegen": ["cargo run --bin gen"]
  },
  "commit": {
    "types": ["feat", "fix", "refactor", "chore"],
    "scopes": ["core", "cli"],
    "protectedBranches": ["main"],
    "branchPattern": "<type>/<short-desc>"
  }
}
```

Detect anything the file leaves out, and never invent vocabulary:

- **Commit format** — read from commitlint config, `.czrc`, CONTRIBUTING,
  `CLAUDE.md`, `AGENTS.md`, or the style of recent `git log`. Mirror the existing
  type and scope vocabulary and the scope-omission rules.
- **Branch names** — infer the pattern from existing branches and open PRs; if
  none is clear, use `<type>/<short-desc>`.
- **The gate** — the repo's format, lint, type-check, build, and test commands,
  from `package.json` scripts, `Makefile`, `Justfile`, the CI workflow,
  CONTRIBUTING, `CLAUDE.md`, or the language's conventional tooling. Split what
  you find into a fast half (format, lint, types, build) and a full half (the
  test suite). When detection is ambiguous, say what you settled on before
  running it, and offer to record it in `.claude/ez.json`.
- **Protected branches** — `main` and `master` unless the config says otherwise.

## Workflow

### Phase 0 — Preflight
1. `git status --porcelain=v1`; capture modified, added, deleted, renamed, AND
   untracked (`??`) paths. Untracked files are easy to miss — include them. If
   the tree has nothing to commit, say so and stop.
2. Confirm the base branch (`git branch --show-current`) is the intended root.
   Don't assume `main`.
3. **If that branch is protected, create and switch to a working branch now** —
   before anything is staged: `git switch -c <type>/<short-desc>`, per the
   branch pattern, inferred from the dominant change. This default name needs no
   approval; the guardrail is that the protected branch stays untouched, not
   that the name is perfect. If Phase 2 later decides a stack is the right
   shape, this becomes the stack's first branch — don't redo it. Skip this step
   only when already on an unprotected branch.
4. If the index already has staged changes, propose starting clean:
   `git restore --staged .`.
5. Run the fast gate now; reserve the full gate for branch tips (Phase 5).
   Report results; a failure may deserve its own fix commit.
6. Skim the diff to gauge scope (`git diff --stat` plus untracked files). Judge
   scope by logical concerns, not line totals.

### Phase 1 — Survey the change
Read every change and group it by **logical concern, not by file**: per group,
what it does, the files and hunks involved, and what it depends on. This is the
raw material for slicing, not a code review; the critical read is
`/ez:review`'s. If `.git/info/ez-review.md` is missing or headed with another
branch, say in one line that the tree has not been reviewed, and go on.

Flag only what changes how the tree gets sliced, and carry each flag to the
slice it touches in Phase 4:
- **Stray changes** — unrelated edits, debug prints, leftover scaffolding. Each
  wants its own commit or no commit; ask which.
- **Weakened gates** — a build, test, or CI edit that removes, skips, or softens
  a check. It lands as its own commit, never inside a feature slice, and the
  user rules on it.
- **Missing tests** — new behavior with no test. Tests ride with the code they
  cover (Phase 3), so name the gap before the plan is set.

Present the groups and any flags BEFORE proposing how to commit.

### Phase 2 — Choose the branch strategy
Default to a **single branch**. Propose a stack of branches only when scope
warrants it — multiple independent themes, changes with different risk or review
audiences, or one theme large enough to be hard to review as a unit (~400
changed lines is a soft nudge to reconsider, never a hard rule; coherence
decides). If splitting, design a linear stack off the base (base → b1 → b2 → …),
each branch one theme describable in a sentence and depending only on branches
below it (foundation → feature → wiring/polish). Present the plan, which
change-groups land where, and proposed branch names. Get explicit approval on
the split AND the names before touching git.

### Phase 3 — Granular commit plan for the current branch
Propose an ordered commit sequence that builds understanding:
scaffolding/types → core logic → wiring. Each commit = one atomic, logical
change (split by purpose, not size). List as `n. <type>(<scope>): <desc>` with a
one-line note on contents.

**Tests ride with the implementation they cover** — never a trailing test-only
commit. If that makes a commit too big to read, the feature wants a finer slice,
not a deferred test.

**Coverage check:** before executing, confirm the plan accounts for every changed
and untracked hunk **exactly once** — nothing dropped, nothing assigned to two
commits. Show the user the change→commit mapping and reconcile it against the
Phase 0 status. Get a nod, then execute.

### Phase 4 — Execute one commit at a time
For each planned commit:
1. Stage only that commit's changes (see Staging).
2. Show `git diff --staged --stat`, and name in one line any Phase 1 flag that
   touches this slice.
3. Present the commit message as a standalone code block so it stands out, and
   nothing else — no commit command, no instructions:
   ```
   feat(cli): add the --watch flag
   ```
4. Run `git commit -m "<message>"` and move to the next slice, reporting what
   landed as you go. The user can still re-slice after the fact — an unpushed
   commit is cheap to redo, so keep moving rather than pausing for a nod.
5. **Manual mode instead:** before the message, show the key hunks for this
   slice and say what the commit does, why it is a coherent unit, and what to
   scrutinize. Then STOP at step 4. The user may re-slice, move a hunk, split,
   or reword — adjust the index (`git restore --staged <path>` to back out).
   When they say "next"/"done", they have committed: re-run `git status` and
   `git log --oneline -1` to resync to what actually landed (they may have
   tweaked or amended), then continue from the true state.

### Phase 5 — Finish the branch, then chain
When a branch's commits are done, run the full gate at the BRANCH level
(individual commits need not build, but the branch tip must). Run the fast and
full halves together. When the branch touches generated files, add the repo's
codegen step — `gate.codegen` from the config, or the command the repo's CI runs
— followed by `git diff --exit-code`, so a stale artifact fails here rather than
in review.

Run it directly when the tree is now clean. For a **stack**, later-branch changes
are still uncommitted, so never stash the live work — verify the tip in a
throwaway worktree (`git worktree add --detach <tmp> HEAD`, run the gate there,
`git worktree remove <tmp>`). A fresh worktree may need its own dependency
install; if that is heavy, say so and offer to skip.

A failing gate may warrant an extra fix commit on the branch. If more branches
remain, create the next off the current tip so the stack chains
(`git switch -c <approved-name>`, which carries uncommitted changes forward and
touches no remote), and return to Phase 3.

When `git status` is clean, summarize the finished branch(es) and close by naming
`/ez:pr` as the next step. Don't propose a PR title here; titling the branch is
that skill's job, and it does it against the merge target rather than guessing at
one.

## Staging mechanics
Prefer staging **whole files** — most commits map cleanly to a set of files:
`git add <paths>`. To split one file across commits, stage by hunk with
`git add -p`, or apply a trimmed patch to the index
(`git diff -- <file> > .git/info/ez.patch`, keep only this commit's hunks, then
`git apply --cached .git/info/ez.patch`) — whichever the runtime supports. For a
new or untracked file, `git add -N <file>` first so it becomes diffable. Unstage
with `git restore --staged <path>`. The patch lives under `.git/`, so it never
shows in `git status`.

## Resilience
For large or multi-branch work, keep a checklist at `.git/info/ez-commit-plan.md`
(branch plan + per-branch commit list + current position + any Phase 1 flag
left unresolved) and update it as commits land, so the plan survives a long
session and `/ez:pr` can pick the concerns up for the PR body. Head it with the
branch it belongs to. `/ez:pr` deletes it once the PR is open; leave it in place
until then.

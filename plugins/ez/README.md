# ez — a portable commit chain for Claude Code and Codex

Nine skills that carry a change from a chosen direction to open pull request
and tidy the checkout once it merges, in any git repository and any language. Nothing here is language-specific;
per-repository detail lives in that repo's own `.claude/ez.json` and
`.claude/rules/comments.md`.

```
.claude-plugin/plugin.json
skills/plan/              map what exists, propose and evaluate directions, build the one picked
skills/review/            critically review a change; findings only, never edits
skills/fix/               rule on each finding, propose and pressure-test the fix, apply on approval
skills/comments/          audit and trim the comments in changed files
skills/commit/            land the tree as ordered atomic commits
skills/pr/                push the branch and open the PR
skills/publish/           comments, prose, commit, and PR in one run
skills/clean/             after the merge: back to the default branch, landed branches and leftovers gone
skills/no-ai-slop/        prose judgment; vendored, see UPSTREAM.md
agents/review-finder.md   read-only finder /ez:review max fans out to, one per angle
agents/review-verifier.md read-only verifier /ez:review max runs on each candidate
agents/fix-proposer.md    read-only proposer /ez:fix runs per finding: verdict first, then candidates
agents/fix-tester.md      read-only tester /ez:fix runs per finding: breaks candidates, picks one
agents/plan-scout.md      read-only scout /ez:plan runs per focus: maps what exists, never proposes
agents/plan-proposer.md   proposer /ez:plan runs once a round, read-only apart from scratch probes: directions against the bar
agents/plan-evaluator.md  evaluator /ez:plan runs once a round, read-only apart from scratch probes: own reuse search, verdicts, rank
```

## The chain

The chain never advances on its own. The model may start `/ez:plan` when it
sets out to plan work, and the run stops for the user's pick. The one
exception is an unattended `/ez:plan` run, which is one started with `--auto`
or, in Claude Code, one nobody can answer, such as a headless run or a
subagent. It builds unasked only when exactly one direction passed and it
commits to nothing one-way. Every
later step runs when the user asks for it, or through `/ez:publish`, which
runs steps 4 to 6 in a row when the user invokes it or asks the model to
publish the branch.

1. `/ez:plan <issue | problem> [notes]` — plans the work before any of it is
   built. Read-only scouts map what already exists in the codebase, and in
   published references when the problem needs them. A fresh-context agent
   proposes one to three directions that differ in mechanism, against a
   written bar (solves it, builds on what exists, at the right layer (module
   and dependency direction included), as small as done allows, breaks
   nothing, decides only what it must, verifiable). A second fresh agent,
   which never sees the scouts' map, searches the code on its own and judges
   each direction; when none passes, one more round runs. It prints the
   directions that pass, ranked, and stops. On the pick it writes the plan
   to `<git-dir>/ez-plan-<key>.md` and
   builds it in the same session. The model may invoke it as well as the user. Every
   run pays for several agents, but nothing is built until the user picks,
   except in the unattended run described above.
   `skills/plan/SOURCES.md` records the evidence behind its shape.
2. `/ez:review [level] [notes]` — critically reviews the change for
   correctness, reliability, performance, simplicity, and fit with the
   codebase, and prints a ranked list of findings, each checked and backed by
   the scenario in which the code misbehaves. The level comes first. It is
   one of `low`, `medium`, `high`, `xhigh` (default), or `max`, the same dial
   as Claude Code's own `/code-review`, and picks the same recipe, from one
   diff pass to a multi-agent find-and-verify run at `max`. The notes are
   free-form: name a PR number, branch, range, or path to review that instead
   of the checked-out branch and its open PR, say what the change is for or
   where to look hardest, or ask for every finding past the cap. Never edits,
   never posts. Run it once more after fixing and it reconciles against the
   last run; a third run is not evidence, since a review that may not drop
   on uncertainty rarely returns nothing on correct code.
   `skills/review/SOURCES.md` records the evidence behind its shape.
3. `/ez:fix [selection] [notes]` — takes the review's findings and, per
   finding, has a fresh-context agent rule on whether it is real and propose
   two or more fixes against a written bar (resolves the scenario, breaks
   nothing, matches a cited neighbor, names a standard only when one exists,
   patches the cause not the symptom and adds no dependency pointing the
   wrong way, as small as it can be, tested the way the repo tests). A second
   fresh agent tries to break each candidate and picks one. It prints the
   plan and stops; on approval it applies the
   approved fixes in order and records in the review's report what it
   refuted, applied, or the user dismissed, and what each applied fix chose
   over what, so the next `/ez:review` checks the fixes, does not raise the
   rest again, and does not argue a decision back. A refuted finding is a
   result. `skills/fix/SOURCES.md` records the evidence behind its shape,
   including why the judgment is never left to the context that wrote the
   fix.
4. `/ez:comments [files]` — audits and rewrites comments in the changed files to
   the house style. Defaults to everything dirty against `HEAD`.
5. `/ez:commit` — sorts the tree and lands it as an ordered series of one-line
   conventional commits, one slice at a time. Pass `manual` to have it stage each
   slice and hand the commit back. It prints its plan and commits without
   waiting for a nod, and stops only for a stack split or a judgment call.
6. `/ez:pr [target]` — pushes the branch and opens a PR titled as a one-line
   conventional commit. Pass `manual` to approve the title and body before
   anything reaches the remote.

`/ez:publish [target]` runs steps 4 to 6 in one go once review and fixes are
done. It runs `/ez:comments` over every file the branch touches, old comments
included, and `/ez:no-ai-slop` over the docs prose the branch wrote, then
`/ez:commit` and `/ez:pr`. Each step keeps its own stops. Its arguments pass
to `/ez:pr`. The model may invoke it as well as the user, but only when asked
to publish; a clean review is not that request.

Stacked branches are supported, through GitHub's `gh stack` extension and no
other tool. `/ez:commit` proposes a stack split, waits for approval of the
split and the branch names, and chains the branches with `gh stack init` and
`gh stack add`. `/ez:pr` opens PRs in the same pass for the branches below
that have none and links them into the stack on GitHub. `/ez:review`,
`/ez:fix`, and `/ez:publish` take the branch below as the base, and the first
two find it only once it has an open PR. Restacking (`gh stack rebase`,
`sync`, `push`, `submit`) stays the user's, because the chain never rewrites
history or force-pushes.

`/ez:clean` runs after the merge, which stays the user's to do. It fetches,
returns the checkout to the default branch and fast-forwards it, and deletes
what the finished work left behind: local branches proven to have landed (a
squash merge counts), clean worktrees whose branch landed, the chain's state
files in the git directory, and scratch files the session made. A worktree
with submodules is removed with `--force`, the only way git allows, once
each submodule proves clean and holds no commit or stash its remote lacks.
Everything else stays and is listed with its reason. It asks one question,
and only when there is something to ask about: untracked files it did not
create, a `picked` plan that may have been built and landed, or a landed
worktree that holds ignored files other than directories a tool provably
generated, such as `.venv/` or `.godot/`. A state file that may describe
those untracked files rides in it. The model may invoke it when the user says
the PR merged or the work is finished.
`skills/clean/SOURCES.md` records the evidence behind its rules.

`/ez:review` also runs usefully after step 6, against the open PR, or against
someone else's PR by number, and again after `/ez:fix`, where it reconciles
against its last report, including the record `/ez:fix` added to it.

`/ez:no-ai-slop` sits under the chain rather than in it. It owns prose judgment —
PR titles and bodies, docs, and the wording inside comments — and `/ez:comments`
and `/ez:pr` invoke it themselves. Invoke it directly to edit any prose, or
with "is this slop?" to audit without rewriting.

Nothing here is enforced by a hook. `/ez:commit` is the way to commit because
each skill says so and because its description says so in every session, not
because a guard denies the alternative.

**A project's own version of any of these wins.** A repo carrying
`.claude/skills/*review*/`, `*fix*/`, `*commit*/`, `*pr*/`, `*comment*/`,
`*clean*/`, or its
own prose skill knows the house conventions; each skill checks for one and
defers. Plugin skills are namespaced, so a repo's `no-ai-slop` and this one
coexist without shadowing.

## Picking a path

A change runs only the steps it needs. The skills' own descriptions draw the
lines:

- **Already decided.** The request says what to edit, the bug's cause is
  known, or the change is a rename. Make the edit, then `/ez:commit` and
  `/ez:pr`. `/ez:commit` says in one line that the tree was not reviewed and
  goes on.
- **Needs a critical read.** The mechanism is settled, but the change should
  be read before it ships. Build it, then `/ez:review` (`xhigh` unless a
  level is given), `/ez:fix`, `/ez:review` once more, and `/ez:publish`. When
  nobody else will read the change before it merges, run `/ez:review max`
  once before the merge. For a solo maintainer that run stands in for the
  second reviewer.
- **Direction still open.** The work is an issue, names no mechanism, could
  be built more than one way, adds a component, or changes a public
  interface, a stored format, or a dependency. Start with `/ez:plan`, whose
  agents are worth paying for here and not on the first path, then go on
  from the review as above.

## Runs with nobody to answer

The chain assumes someone answers its questions. In CI, an `@claude` GitHub
Action, or a scheduled agent, nobody does. `/ez:commit` stops on a judgment
call and never pushes, and `/ez:fix` stops for approval before it writes
anything. Only `/ez:plan` has an unattended mode. In Claude Code a run
nobody can answer is unattended even without `--auto`, and it builds only
under the test described in **The chain**.

A repo that tells agents to commit only through `/ez:commit` gives a CI agent
a rule it cannot follow where the plugin is not installed. Fixing that is the
consuming repo's job, not the plugin's. One repo exempts its CI agent through
claude-code-action, with `claude_args: --append-system-prompt "..."` telling
it to ignore the commit rule and to commit and push itself. The action cannot
install this marketplace from its SSH URL. As of claude-code-action 9fa6885
(2026-11-13), `plugin_marketplaces` accepts only an `https://` URL ending in
`.git` or a local path.

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

Requires `gh` authenticated for `/ez:pr`, for `/ez:plan` to read an issue, and
for `/ez:review` to find a branch's PR; without it the review falls back to the
default branch as base. The `github/gh-stack` extension is optional. Without
it `/ez:commit` chains a stack's branches with `git switch -c` and says the
stack is untracked, and `/ez:pr` opens one PR at a time. It targets the head
of the nearest open PR that is an ancestor of the branch, confirmed first, or
the default branch when none is, and chains the PRs by base without a stack.

```sh
claude plugin marketplace add git@github.com:coffeebeats/claude-ez.git
claude plugin install ez@claude-ez
```

Verify with `claude plugin details ez`, which should list nine skills, seven
agents, and no hooks.

## Running under Codex

Codex installs the plugin from the same marketplace (the root README has the
commands) and loads all nine skills; read `/ez:x` as `$ez:x`. Four things
differ, each from what Codex can and cannot do as of codex-cli 0.156:

- **Agents run on the session model.** A Codex plugin cannot ship agent
  types, so `/ez:plan`, `/ez:fix`, and `/ez:review max` spawn a fresh agent
  with no forked context and open its task with the role's file from
  `agents/`. The file's `effort` carries over; its `model` does not, so the
  per-role cost pins hold only in Claude Code. Nor are the `tools:` lists
  enforced: the agents stay read-only because their instructions say so.
- **Agent slots are few.** `/ez:review max` spawns more agents than Codex
  runs at once, so a refused spawn waits for a running agent to finish and
  retries, and where Codex offers a close tool the skills close each agent
  when done.
- **The sandbox asks first.** Codex keeps `.git` read-only and the network
  off by default, so state-file writes in the git directory, commits,
  worktrees, `git fetch` and `push`, and `gh` each ask for approval.
- **Questions are plain text.** Codex has no question tool outside Plan mode,
  so a stop is a question in the reply and the turn ends. `/ez:plan` builds
  unasked only under `--auto`.

The `/ez:review` levels were measured on Claude models. A Codex model runs
the same table, unmeasured.

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
resolves the plugin to its commit SHA, so `claude plugin update ez` and
auto-update track the branch, and a running session picks up the new copy at
`/reload-plugins` or its next start. Adding a version means bumping it on every
release or clients silently keep the old copy. To try an edit before it is on
`main`, see the root README.

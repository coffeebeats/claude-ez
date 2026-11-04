# ez — a portable commit chain for Claude Code

Six skills that carry a change from working tree to open pull request, in any
git repository and any language. Nothing here is language-specific; per-repository
detail lives in that repo's own `.claude/ez.json` and `.claude/rules/comments.md`.

```
.claude-plugin/plugin.json
skills/review/            critically review a change; findings only, never edits
skills/fix/               rule on each finding, propose and pressure-test the fix, apply on approval
skills/comments/          audit and trim the comments in changed files
skills/commit/            land the tree as ordered atomic commits
skills/pr/                push the branch and open the PR
skills/no-ai-slop/        prose judgment; vendored, see UPSTREAM.md
agents/review-finder.md   read-only finder /ez:review max fans out to, one per angle
agents/review-verifier.md read-only verifier /ez:review max runs on each candidate
agents/fix-proposer.md    read-only proposer /ez:fix runs per finding: verdict first, then candidates
agents/fix-tester.md      read-only tester /ez:fix runs per finding: breaks candidates, picks one
```

## The chain

Every step is user-invoked; the chain never advances on its own.

1. `/ez:review [level] [notes]` — critically reviews the change for
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
2. `/ez:fix [selection] [notes]` — takes the review's findings and, per
   finding, has a fresh-context agent rule on whether it is real and propose
   two or more fixes against a written bar (resolves the scenario, breaks
   nothing, matches a cited neighbor, names a standard only when one exists,
   patches the cause not the symptom, as small as it can be, tested the way
   the repo tests). A second fresh agent tries to break each candidate and
   picks one. It prints the plan and stops; on approval it applies the
   approved fixes in order and records in the review's report what it
   refuted, applied, or the user dismissed, so the next `/ez:review` checks
   the fixes and does not raise the rest again. A refuted finding is a
   result. `skills/fix/SOURCES.md` records the evidence behind its shape,
   including why the judgment is never left to the context that wrote the
   fix.
3. `/ez:comments [files]` — audits and rewrites comments in the changed files to
   the house style. Defaults to everything dirty against `HEAD`.
4. `/ez:commit` — sorts the tree and lands it as an ordered series of one-line
   conventional commits, one slice at a time. Pass `manual` to have it stage each
   slice and hand the commit back.
5. `/ez:pr [target]` — pushes the branch and opens a PR titled as a one-line
   conventional commit. Pass `manual` to approve the title and body before
   anything reaches the remote.

`/ez:review` also runs usefully after step 5, against the open PR, or against
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
`.claude/skills/*review*/`, `*fix*/`, `*commit*/`, `*pr*/`, `*comment*/`, or its
own prose skill knows the house conventions; each skill checks for one and
defers. Plugin skills are namespaced, so a repo's `no-ai-slop` and this one
coexist without shadowing.

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

Requires `gh` authenticated for `/ez:pr`, and for `/ez:review` to find a
branch's PR; without it the review falls back to the default branch as base.

```sh
claude plugin marketplace add git@github.com:coffeebeats/claude-ez.git
claude plugin install ez@claude-ez
```

Verify with `claude plugin details ez`, which should list six skills, four
agents, and no hooks.

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

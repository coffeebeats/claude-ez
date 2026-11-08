---
name: review
description: >-
  Critically reviews a change — the checked-out branch and its open PR by
  default, or a PR number, branch, commit range, or path named in free-form
  notes, which can also say what the change is for and where to look
  hardest — for correctness, reliability, performance, simplicity, and fit
  with the codebase's own patterns, and prints a ranked list of findings each
  anchored to a line and backed by the concrete scenario in which the code
  misbehaves; an empty report is a valid result. A leading effort word (low,
  medium, high, xhigh, max) sets how much the review spends, from one diff
  pass to a multi-agent find-and-verify run, on the same recipe table as
  Claude Code's own /code-review. Never edits and never posts to the PR. Run
  it again after fixing and it reconciles against the previous run on the
  same branch in the same checkout. Works in any git repository and any
  language. Use this whenever a change needs a critical read before it ships —
  invoked by the user via /ez:review, or by the model when asked to review,
  critique, or sanity-check code.
argument-hint: "[low|medium|high|xhigh|max] [notes: pr#, branch, range, path, or focus]"
allowed-tools: Bash, Read, Grep, Glob, Agent, Write
---

# ez:review

Find what is wrong with a change before a human reviewer or production does.
The output is a short ranked list of findings a maintainer would act on, each
backed by a scenario in which the code misbehaves. A tour of the diff, praise,
and style notes are not output.

**A project skill wins.** If this repository carries its own review skill —
`.claude/skills/*review*/SKILL.md` — that skill knows the house bar and this one
does not. Name it and stop rather than running.

## Hard rules — never violate

- NEVER edit the working tree, stage, commit, switch branches, or touch a
  remote. The one file you write is the report at `<git-dir>/ez-review.md`
  (Phase 0 resolves `<git-dir>`). Fixing belongs to `/ez:fix`, after the user
  has read the report.
- NEVER post to the PR, open an issue, or comment anywhere. The report goes to
  the user, in the terminal.
- Every finding cites a `path:line` and carries a **failure scenario** — the
  input, state, or timing that makes the code misbehave, and what it does
  wrong. A cleanup finding carries a **concrete cost** instead. A candidate you
  cannot back up with one of those is dropped, not hedged.
- **An empty report is a valid outcome.** Finding nothing after a real search
  is a result; inventing a finding to fill the list is a failure. No level
  below sets a floor, and none of them pads.
- Judge the change against what it is trying to do. A behavior change the PR
  body or commit message asks for is not a bug.

## The bar

A finding is something the author would fix if they knew about it. All of
these hold at every level:

- It is **introduced or re-exposed by this change**, not merely present in the
  file. A problem on lines the change did not touch is out of scope unless the
  change makes it worse or newly reachable; name at most one such thing,
  marked pre-existing, and only when it is serious.
- It is **demonstrable from the code**, with a line you can cite, not an
  inference from a name or a guess about intent. "May disrupt another part of
  the codebase" is not a finding; the affected caller, named, is.
- It is **discrete and actionable** — one thing, at one place, with a fix the
  author could make.
- It does **not demand rigor the rest of the codebase lacks**. Asking for a
  pattern the repo uses nowhere is advice, not a finding.
- It is **not what tooling reports.** The compiler, type-checker, linter, and
  formatter run in CI; a missing import, a type error, a format nit is theirs.
  Running the gate is `/ez:commit`'s job, not this skill's. A single existing
  test or a one-line scratch check that confirms or refutes a candidate is
  fine when it is side-effect free.
- It is **not a nit a senior engineer would let pass** — naming taste, a
  comment's wording, a style preference nobody wrote down.

The bar says what counts. Which way to lean on a candidate you half believe
is set by the level, below.

## Inputs

`/ez:review [level] [notes]`. Both are optional.

The level is the first word when that word is `low`, `medium`, `high`,
`xhigh`, or `max`; with none given it is `xhigh`. Only the first word is read
this way, so a level word later in the notes is prose.

Everything after the level is notes, in the user's own words. Most runs have
none, and the review covers the checked-out branch. Read the notes for three
things and say which you found:

- **A target** — a PR number or URL, a branch, a commit range or revision, or
  paths that the notes ask to review. Resolve it per the cases below. A name
  the notes only mention, as in "the retry change in `src/net.ts` is
  deliberate" or "look hardest at `auth/`", is guidance, and so is a bare
  word that names nothing; the scope stays the checked-out branch. When it is
  unclear which the user meant, read it as guidance. When the notes ask to
  review a PR, branch, range, or path that does not exist, say so and stop
  rather than guess.
- **Guidance** — what the change is for, where to look hardest, what to leave
  alone. Intent stated here counts as the PR body does. Focus decides where
  the angles dig first; it switches none of them off unless the notes ask to
  look only at one thing. Guidance never lowers the bar and never sets a
  floor: "find at least five" gets no padding.
- **A request for every finding**, which lifts the level's cap.

Resolve the target in this order and say which case applied:

- **No target in the notes.** The current branch. Its base is the base of its
  open PR (`gh pr view --json baseRefName,number,title,body,url`), else the
  branch below it in a stack (the head of the open PR that is an ancestor of
  HEAD, nearest first, per `git merge-base --is-ancestor`), else the repo's
  default branch (`git symbolic-ref refs/remotes/origin/HEAD`, falling back
  to `main`/`master`). The scope is `git diff <merge-base>...HEAD` **plus** the
  working tree — `git diff HEAD` and every untracked file — because the review
  usually runs before the commit. On the default branch itself the scope is the
  working tree alone; if that is clean, say there is nothing to review and stop.
- **A PR number, `#123`, or a PR URL.** `gh pr view <n> --json
  headRefName,headRefOid,baseRefName,title,body,url,reviews,comments`. When
  its head is the current branch, review as above. Otherwise never switch
  branches: `git fetch origin pull/<n>/head` and read the code from a detached
  throwaway worktree (`git worktree add --detach <tmp> FETCH_HEAD`, removed
  with `git worktree remove <tmp>` at the end). The scope is the merge-base of
  base and head, to head.
- **A branch name** that exists locally or on `origin`. As the PR case: current
  branch → in place; otherwise a worktree at its tip. Its base is its open PR's
  base if it has one, else the default branch.
- **A commit range** (`a..b`, `a...b`) **or a single revision** (`HEAD~2`, a
  SHA). That range, from the current checkout when its tip is HEAD, otherwise
  from a worktree at the tip.
- **One or more paths** that exist. A whole-file review of those files as they
  stand, with no diff: treat every line as changed, skip the removed-behavior
  angle, and say that the review is of the file rather than of a change.

## Levels

The level is how much the review may spend, and it picks the recipe. The table
is the one Claude Code's bundled `/code-review` runs for Opus-class models,
cell for cell, so a bare `/ez:review` does what a bare `/code-review` does
(`SOURCES.md` records where it comes from). Below `max` nothing is delegated:
the angles, the check, and the sweep all run in this context.

| Level | Recipe | Angles | Per angle | Lean | Cap |
|---|---|---|---|---|---|
| `low` | one diff pass, hunks only, no file reads | — | — | precision | 4 |
| `medium` | angles inline → dedup → self-check | A B C E G H I J | 6 | precision | 10 |
| `high` | angles inline → dedup → self-check | A B C E G H I J | 6 | recall | 10 |
| `xhigh` | angles inline → dedup → self-check → sweep | A–K | 8 | recall | 15 |
| `max` | finder agents → dedup → verifier agents → sweep | A–K | 8 | recall | 15 |

**Precision** means every finding you surface should be one a maintainer would
act on: a half-believed candidate still goes to the check, and the check drops
what you cannot stand behind. **Recall** means catching real bugs matters more
than avoiding false positives — a missed bug ships — so err on the side of
surfacing and do not drop on uncertainty: a candidate with a nameable scenario
survives the check unless the code refutes it.

**Smaller models.** If you are a Sonnet-class or smaller model, `medium` and
above run the `max` recipe with the chosen level's angles, candidates per
angle, lean, and cap. That is the recipe measured for them.

**Agents.** Only the `max` recipe spawns them. Spawn every agent through the
Agent tool with its `effort` set to the level word. If the Agent tool is
unavailable, run the `xhigh` recipe at the chosen level's cap and say so in
the report. If the tool is there but does not offer the `ez:review-finder` or
`ez:review-verifier` type, dispatch `general-purpose` with the contents of
`${CLAUDE_PLUGIN_ROOT}/agents/review-finder.md` or `review-verifier.md` as
the opening of its prompt. An agent reads; it never edits.

## What the review covers

Five dimensions, each with the lenses in Phase 2. They are lenses on the same
code, not sections of the report: a finding belongs to whichever dimension
names its cost, and the report is ranked by severity, not grouped by dimension.

| Dimension | Asks |
|---|---|
| **Correctness** | Does the code do what the change intends, for every input, state, and ordering it can meet? |
| **Reliability** | What happens when something fails, races, or arrives from outside, and does the caller find out? |
| **Performance** | Does the change do clearly wasted work, or put work on a path that cannot afford it? |
| **Simplicity** | Is there a plainly simpler or existing form that does the same job, and is the fix at the right depth? |
| **Fit** | Does it do things the way this codebase does them, and where it departs from the language's idiom, does the departure cost anything? |

"Best practice" on its own is not a finding. A departure from one is a finding
when you can name what it costs — a bug it invites, a reader it misleads, a
tool it breaks — and the repo does not already do it the same way everywhere.

## Workflow

### Phase 0 — Resolve the target
Before anything else, run `git rev-parse --absolute-git-dir` in the checkout the
skill was invoked in and record the result as `<git-dir>`. It is the directory
git keeps for this checkout alone: `.git` in the main checkout,
`<common .git>/worktrees/<id>` in a linked worktree. The report lives there and
nowhere else, so a session in another worktree cannot replace it. Keep using
this value for the whole run, and never resolve it from inside a throwaway
worktree: `git worktree remove` deletes that worktree's private directory and
whatever is in it. Quote it in shell commands, since the path may contain
spaces.

1. `git status --porcelain=v1` and `git branch --show-current`.
2. Resolve the target per **Inputs**. Record the base, the head SHA, the
   level, any guidance from the notes, and whether the scope includes
   uncommitted work.
3. Gather the scope: `git diff --stat` and the unified diff for the range, the
   working-tree diff, and the list of untracked files.
4. Read the repo layer: `CLAUDE.md`, `AGENTS.md`, any `CLAUDE.md` in a
   directory that is an ancestor of a changed file, `.claude/rules/`, and
   `.claude/ez.json`. These are the rules the **Fit** dimension can quote.

At `low`, step 4 is skipped and the rest collapses into as few calls as
resolving the target allows — ideally one, for the diff.

### Phase 1 — Understand the change
Skipped at `low`. Otherwise, before hunting for faults, state in two or three
lines what the change is trying to do and how. Take it from the notes, the PR
title and body, the commit messages (`git log --format=%B <base>..<head>`),
and the diff itself. Where the stated intent and the diff disagree, that gap
is the first candidate.

If the target is a PR with review comments, read them. A point a reviewer
already raised and the author already answered is not a finding; a point
raised and ignored is one, and you may say so.

Then read for context, not just the hunks: the enclosing function of every
hunk, and the callers of every function whose signature or contract changed
(Grep the symbol). Bugs in unchanged lines of a touched function are in scope —
the change re-exposes them or fails to fix them.

### Phase 2 — Find candidates

**At `low`:** one pass over the hunks, skipping test and fixture files, with no
file reads and no agents. Phase 1 does not run, so take the notes' guidance
here: skip what they leave alone, and judge each hunk against the intent they
state. Flag runtime-correctness bugs visible from the hunk
alone — an inverted or wrong condition, off-by-one, a null or missing value
where adjacent lines show it can be absent, a removed guard, a falsy-zero
check, a missing `await`, wrong-variable copy-paste, an error swallowed in a
catch that should propagate — plus, still from the hunk alone, new code that
duplicates a helper visible in the diff context, and dead code, debug output,
or scaffolding the diff leaves behind. Nothing else: not style, naming,
performance, tests, or anything outside the hunk. Each candidate carries its
`path:line` and scenario. Then go straight to Phase 5.

**Otherwise,** work through the level's angles, in order. Each angle yields
**candidates**: `path:line`, the dimension, a one-sentence claim, and the
failure scenario or concrete cost. Record each candidate as soon as an angle
produces it. If two angles flag the same line for different reasons, record
both; one angle's conclusions never suppress another's. Do not drop a
half-believed candidate. Phase 3 decides, and candidates silently dropped here
are the main way real bugs are missed. Up to the level's count per angle; past
that, keep the most severe.

The angles say where to look, not what to find. They exist so the search is
the same every run; an angle with nothing behind it stays empty, and most do
on a clean change.

Below `max`, run the angles yourself, in sequence, in this context. At `max`,
dispatch one `ez:review-finder` per angle via the Agent tool, in parallel,
each given the diff (or the range and worktree path), the Phase 1 summary, the
guidance from the notes, the repo rules, and the text of its angle; pool what
they return.

#### Correctness
- **A — Line by line.** Read every hunk, then the function around it. For each
  changed line ask what input, state, timing, or platform makes it wrong:
  inverted or incomplete conditions, off-by-one, a null or missing value where
  adjacent lines show it can be absent, a missing `await` or unjoined task,
  falsy-zero or empty-string treated as missing, wrong-variable copy-paste,
  unescaped regex metacharacters, integer overflow and float equality,
  timezone and encoding assumptions.
- **B — Removed behavior.** For every line the change deletes or replaces, name
  the invariant or behavior it enforced, then find where the new code
  re-establishes it. A removed guard, a dropped error path, a narrowed
  validation, or a deleted test that covered a real case is a candidate when
  you cannot find its replacement.
- **C — Cross-file.** For each function the change alters, check every caller:
  a new precondition, a changed return shape, a new exception, a new ordering
  dependency. Check callees too — does another hunk in the same change make a
  call unsafe? For a type that wraps another (cache, proxy, adapter), check
  every method routes to the wrapped instance and that it forwards what callers
  actually use.
- **D — Language pitfalls.** The classic traps of the diff's language and
  framework, where the change introduces one: mutable default arguments and
  late-binding closures, loop-variable capture, `==` coercion, nil-map writes,
  shared mutable state across async boundaries, iterator invalidation,
  lifecycle methods that must call super, ORM lazy loads in loops.

#### Reliability
- **E — Failure paths.** For every error path the change adds or touches: what
  happens on failure, and does the caller find out? Errors swallowed or logged
  and forgotten, a broad catch that hides unrelated failures, a fallback that
  masks the problem it falls back from, partial writes with no rollback, a
  retry without a bound, a missing timeout, a resource, lock, handle, or
  subscription not released on every path, work that is not idempotent where
  it will be retried, ordering or concurrency the code assumes but nothing
  enforces.
- **F — Trust boundaries.** Where input from outside the process — a user, the
  network, a file, the environment, arguments — newly reaches a sink: shell,
  SQL, path, HTML, deserialization, eval, a permission check, a log line that
  would carry a secret. Only where the change takes part; a pre-existing hole
  the change did not widen is out of scope.

#### Performance
- **G — Wasted work.** Redundant computation or repeated I/O the change adds,
  N+1 queries, independent operations run in sequence, allocation or
  regex compilation in a hot loop, blocking work on startup or a hot path,
  unbounded growth of a cache or buffer, a closure capturing a large scope for
  a long-lived object. Name the cheaper form. Clear gaps only, not
  micro-optimizations.

#### Simplicity
- **H — Reuse and simplification.** New code that re-implements a helper the
  codebase already has (Grep shared and utility modules and the files next to
  the change) and name the helper, and a raw primitive called where the
  codebase already wraps it, naming the wrapper. Redundant or derivable state,
  copy-paste with slight variation, deep nesting, an abstraction with one use,
  dead code, debug output, or scaffolding the change leaves behind. Name the
  simpler form that does the same job.
- **I — Altitude.** Does the change fix the cause at the right depth, or patch
  a symptom? A special case layered onto shared infrastructure is a sign the
  fix is too shallow; name the general change it should have been.

#### Fit
- **J — Conventions.** How does neighbouring code do the same thing — the error
  type, the logging call, the naming, the module layout, the test style — and
  does the change do it differently without reason? Where a repo rule exists,
  flag a violation only when you can quote both the rule and the line that
  breaks it. No taste calls, no "spirit of the doc" inferences.
- **K — Test evidence and gates.** Does new behavior have a test, and does that
  test exercise the real code path rather than a hand-built replica or the
  framework? A test that cannot fail, an assertion weakened to pass, a test
  deleted or skipped, setup and teardown that are asymmetric. Note gaps and
  false confidence alike. Then the gate itself, in build, CI, and tool config:
  a check removed, made non-blocking, or filtered so it no longer runs, a lint
  rule or warning silenced, a threshold lowered.

### Phase 3 — Dedup and check
Dedup first: candidates that point at the same line and mechanism collapse to
one, keeping the most concrete failure scenario. Then drop any candidate in
code the notes said to leave alone.

Then every survivor gets one verdict, by the same three-way standard whoever
gives it:

- **CONFIRMED** — the inputs or state that trigger the failure and the wrong
  result can be named, and the line quoted.
- **PLAUSIBLE** — the mechanism is real but the trigger depends on timing,
  environment, or configuration the code does not pin down. Say what would
  confirm it.
- **REFUTED** — factually wrong (the code does not say that), provably
  impossible (a type, constant, or invariant rules it out), already guarded
  (cite the guard), or pure style with no observable effect.

PLAUSIBLE is the default for a realistic runtime state — a race, a rare but
reachable nil, a boundary the code does not exclude. REFUTED needs evidence
that can be read from the code, never an imagined defense. Keep CONFIRMED and
PLAUSIBLE; drop REFUTED. Under a precision lean, also drop a PLAUSIBLE
candidate you would not put your name to.

**Below `max`,** give the verdict yourself: re-open the cited line and the code
around it and re-check the candidate against the diff before keeping it. Do
not re-judge the whole list from scratch, and under a recall lean do not drop
on uncertainty.

**At `max`,** dispatch one `ez:review-verifier` per candidate via the Agent
tool, in parallel, each given the candidate, the Phase 1 summary, the diff,
the files it cites, and the worktree path if there is one. The verifier
reasons from the code first and rules second and returns exactly one verdict.
Its verdict stands; do not re-argue it.

### Phase 4 — Sweep for gaps
At `xhigh` and `max` only. Take one more pass, in this context, as a fresh
reviewer who holds the checked list and is looking **only** for what is not on
it. Do not re-derive anything already there. The first pass tends to miss:
moved or extracted code that dropped a guard or an anchor; a config default
flipped; a predicate with a side effect; a lock whose scope shrank; a value
computed once that must be computed per call; setup and teardown that are
asymmetric in tests. Up to eight new candidates, each checked as in Phase 3.
If there is nothing, say nothing — the sweep does not pad.

### Phase 5 — Report
1. **Reconcile with the previous run.** If `<git-dir>/ez-review.md` exists and
   its heading names this branch, read it now — after your own findings are
   settled, never before, so it cannot anchor them. Mark each of its findings,
   including those it carried under **Not re-checked**,
   **resolved** (the code changed and the scenario no longer holds — read the
   line to confirm), **still open** (it is in this run's list; say which
   number), or **withdrawn** (this run refuted it; say why in a clause). A
   finding that this run did not look for, because the previous run was at a
   higher level or this run's notes narrowed it, is neither; it is **not
   re-checked**, and step 5 carries it forward. Mark this run's remaining
   findings **new**. A file headed with another branch is stale; ignore and
   overwrite it.

   Then the rulings, still after your findings are settled: each entry of
   that report's `Fix record:` marked `refuted`, `pre-existing`, or
   `dismissed`, and each such ruling it carried under `Upheld:`. A ruling
   quotes the line its finding sat on and, unless it is a dismissal, the line
   that decided it. A finding of this run is the ruling's when it names the same
   mechanism on a line that reads as the first quote, and no other line of
   that file reads the same: a pasted copy is a new instance, not the one
   ruled on. It is dropped and counted as **upheld** while the deciding line
   still reads as quoted. When that line has changed, or you can quote what
   the evidence missed, it stays, marked **reopened** with what changed or
   what was missed, in place of new or still open. A user's dismissal counts
   the way intent in the PR body does and is not re-argued. An entry marked
   `applied` is evidence for **resolved**; read the line to confirm, and if
   the scenario still holds, the finding stays, marked **reopened** (the fix
   did not close it). `planned` entries are progress on that report and end
   with it.

   Then the decisions: the one an `applied` entry records, whether its
   finding is now resolved or carried as not re-checked; each `decided`
   entry; each `decided` entry carried under `Upheld:`; and each `chosen`
   entry there, a fix's decision step 5 carried. A decision names what was
   chosen and the alternatives rejected, leaving the code as it was among
   them, each with its reason, and quotes lines of the chosen code. The
   ruling match above does not apply to it. A finding of this run is the
   decision's when its remedy is one of the rejected alternatives, or it
   names the failure of the finding a `decided` entry answers, and it sits
   on, or argues about, a line that reads as one of the decision's quotes,
   on one line of its file only. It stays, marked **reopened** with what
   changed or what was missed, when a quoted line has changed; and, unless
   it names a `decided` entry's own failure, which a user's decision holds
   as a dismissal does, when you can quote evidence other than its own
   scenario that the reason did not weigh, or when its scenario can be
   closed in full while keeping what the decision chose, as a caller a fix
   broke can be mended without reverting the fix; it then stays with that
   remedy in place of the rejected one. Otherwise it is dropped and counted
   as **held**: only a rejected alternative would close its scenario, so the
   failure is the price the decision weighed in rejecting it, as giving up
   on an outage is the price of a retry cap. A decision settles a trade-off,
   not a defect in carrying it out. A user's decision is not re-argued on
   the trade-off it names. A decision whose `applied` entry this run marks
   reopened retires with it. The record suppresses; it never seeds.
2. Rank by severity: correctness and reliability outrank performance,
   simplicity, and fit when the cap forces a cut; within a dimension,
   CONFIRMED outranks PLAUSIBLE, and a failure users would hit outranks one
   they would not.
3. **At most the level's cap** unless the notes asked for every finding. Say
   how many were cut and from which dimension.
4. Print the report in this shape and no other:

   ```markdown
   ## Review — <branch or target> @ <short sha> · <level> · <n> files, +<a> −<d>

   Notes: <the guidance taken from the notes, in one line>.       (if any)
   <two or three lines: what the change does, from Phase 1; omitted at low>

   1. `path/to/file.ext:123` · correctness · CONFIRMED · new
      <one-sentence claim>. Fails when <scenario → wrong result>.
   2. `path/to/other.ext:40` · reliability · PLAUSIBLE · still open (was #3)
      <claim>. <scenario>. Confirm by <what the check named>.
   3. `path/to/third.ext:88` · correctness · CONFIRMED · reopened (<what changed>)
      <claim>. Fails when <scenario → wrong result>.

   Pre-existing, not counted: `path:line` — <one line>.          (if any)
   Previous run: 2 resolved, 1 still open, 1 withdrawn (<why>),
   1 not re-checked.                                              (if any)
   Rulings: 2 upheld, 1 held, 1 reopened, 1 retired.              (if any)
   Clean: <the angles that ran and produced nothing, as one line>.
   Skipped: <angles and code the notes switched off or left alone>. (if any)
   ```

   With nothing to report, the list is replaced by the line
   `No findings. Checked <the dimensions that ran> against <scope>.`, the
   Skipped line when the notes narrowed the run, and one line naming any
   residual risk the review could not settle by reading. No praise section,
   no summary of the diff beyond the Phase 1 lines, no severity labels beyond
   the verdict, no code blocks longer than three lines.
5. Write the same report to `<git-dir>/ez-review.md`, the invoking checkout's
   even when the code was read from a throwaway worktree, headed with the
   branch, head SHA, and level, replacing whatever was there; Read it first
   if step 1 did not, since the Write tool refuses to overwrite a file this
   session has not read. Append every finding step 1 marked not re-checked,
   copied unchanged but numbered on from this run's last, under a
   `Not re-checked:` heading, so a narrowed run never erases what it did not
   look at; leave out one whose ruling or `decided` entry the next two
   sentences carry, since that stands for it. Then append under `Upheld:`
   every ruling that still holds, with its file, mechanism, status, quotes,
   and a dismissal's reason:
   one this run did not reopen, whose file is in this run's scope, and whose
   quoted lines still read as quoted, the first on one line of its file only,
   whether or not this run raised its finding; and, unchanged, one in code
   this run did not look at because the notes switched it off or targeted
   other paths or another range. Every decision that holds by the same test
   (a finding reopened with a remedy that keeps the choice leaves it
   standing), each of its quotes on one line of its file only, goes there
   too, as a
   `decided` entry, or a `chosen` one when an `applied` line recorded it or
   it was carried as `chosen`, with its file, the failure it answers, chosen,
   rejected, reasons, and quotes; only a user's decision is `decided`. The
   rest retire, so a ruling lasts as long as the change it was about. Last,
   when a finding carried as not re-checked has a `Fix record:` line marked
   `applied`, copy that line, renumbered with the finding and with its
   decision, under a `Fix record:` heading, so `/ez:fix`, `/ez:commit`, and
   `/ez:pr` still read that fix as unreviewed; its decision rides there
   rather than under `Upheld:`. The file stays out of `git status`. Remove
   any worktree Phase 0 created.
6. Close with one line naming the next step. After a first run with findings:
   `/ez:fix`, which rules on each finding in a fresh context, proposes and
   pressure-tests the fix, and applies what the user approves; then
   `/ez:review` once more so step 1 marks each prior finding and catches what
   the fixes introduced. After that second run: `/ez:commit` when the tree is
   dirty, `/ez:pr` when the branch is committed and clean, with `/ez:review
   max` once before a merge nobody else will read. When this run marked any
   finding not re-checked, the next step is instead `/ez:review` without the
   narrowing notes, whichever run this is. Otherwise a third run is not
   evidence; a reviewer told not to drop on uncertainty will rarely return
   nothing on correct code, and the stop condition is that nothing is
   CONFIRMED and every PLAUSIBLE has been read and ruled on, not that the list
   is empty.

# Why `/ez:review` is shaped the way it is

Design notes and the references behind them, so the next edit to `SKILL.md`
can be checked against what the evidence says rather than against taste.
Gathered October 2026.

## The shape: find, check, rank, cap

Every serious production reviewer runs the same four stages. Anthropic's
managed Code Review: "Find: several agents examine the diff and the
surrounding code in parallel. Verify: a verification step checks the candidate
findings against the code and filters out false positives. Rank: the findings
that remain are deduplicated and ranked by severity. Post."
Ultrareview: "every reported finding is independently reproduced and
verified." CodeRabbit runs verification scripts against each suggested
comment. OpenAI Codex Security validates in a sandbox.

- <https://code.claude.com/docs/en/code-review>
- <https://code.claude.com/docs/en/ultrareview>
- <https://docs.coderabbit.ai/overview/architecture>
- <https://alignment.openai.com/scaling-code-verification/>

The local `/code-review` skill bundled with Claude Code (its prompt lives in
the binary, not on disk) is the closest relative and the one this skill
copies. Its recipe: gather the diff from `@{upstream}...HEAD` plus the working
tree; run finder *angles* — line by line, removed behavior, cross-file tracer,
language pitfalls, wrapper/proxy, then reuse, simplification, efficiency,
altitude, CLAUDE.md conventions — each returning candidates with `file`,
`line`, `summary`, `failure_scenario`; dedup; a check per candidate returning
CONFIRMED / PLAUSIBLE / REFUTED; at the highest efforts a gap sweep by a fresh
reviewer; a ranked, capped list. Phase 2 through Phase 5 here are that recipe,
widened to the reliability and performance lenses and with the report shape of
the rest of the chain. Two lines from it are load-bearing and carried over:
finders must pass through half-believed candidates because "finders that
silently drop half-believed candidates bypass the verify step and are the
dominant cause of misses," and verdicts are "PLAUSIBLE by default" with
REFUTED "only when constructible from the code."

The open-source predecessor, the `code-review` plugin in Anthropic's official
marketplace, is readable and worth knowing: it only works on a GitHub PR and
posts a comment, so it cannot review a checked-out branch, but its false
positive list (pre-existing issues, looks-wrong-but-correct, nitpicks a senior
engineer would not flag, linter-catchable, general quality concerns absent a
written rule, lint-ignored lines) is the ancestor of **The bar** here.

- <https://github.com/anthropics/claude-code/blob/main/plugins/code-review/commands/code-review.md>

## The level table is Anthropic's, cell for cell

The bundled skill does not run one procedure. It holds a table keyed by model
and then by effort, and several cells are flagged `measuredExternal: true` in
the code — set from an eval rather than by hand. Read out of the binary in
October 2026 (Claude Code 2.1.x):

| Model | low | medium | high | xhigh | max |
|---|---|---|---|---|---|
| default (any model not listed, including Opus 5.5 and Fable 5.1) | 1 diff pass, no check, cap 4 | 8 angles inline, dedup, no agent verify, cap 10 | same as medium | 10 angles inline, dedup, no agent verify, sweep, cap 15 | 10 finder agents × 8, one verifier agent per candidate, sweep, cap 15 |
| Opus 4.8 | as default, plus a soft floor | 8 angles inline, cap 8 | as default | as default | as default |
| Opus 5 | as default | one "careful senior engineer" pass, no angles, cap 15 | same | as default | as default |
| Sonnet 5 | 1 diff pass, soft floor | 8 finder agents × 6, verify, precision lean, cap 8 | 8 finder agents × 6, verify, recall lean, cap 10 | 10 finder agents × 8, verify, sweep, cap 15 | same as xhigh |

Three things in the table decided the shape of **Levels** in `SKILL.md`.

**Agent verification is a Sonnet feature.** For Opus-class models at every
effort below `max` the cells say "do NOT run verifiers; do NOT re-judge" and,
from `high` up, "do NOT drop on uncertainty." A verifier appears for Opus only
at `max`, and for Sonnet from `medium` up. The code does not say why; the
shape implies that a Sonnet finder is noisy enough for a second opinion to pay
and an Opus finder is not, so for Opus the verifier mostly spends tokens and
occasionally refutes a true bug. The first draft of this skill ran the Sonnet
recipe — fan-out above 400 changed lines and a verifier agent on every
candidate — on an Opus-class session. It was replaced by the table.

**The lean is recall, not precision, from `high` up.** The Opus cells at
`medium` and `high` say "catching real bugs matters more than avoiding false
positives. Err on the side of surfacing"; only the Sonnet `medium` cell asks
for precision. The Opus 4.8 cells also carry a soft floor ("target at least N
findings … do not invent to hit the floor"). The skill takes the per-level
lean wording and leaves the floor out: it is the one leading instruction in
the recipe, and a padded finding costs a solo reviewer more than it costs the
eval the floor was tuned on.

**The trend for newer models is less scaffolding.** Opus 5, the one newer
model with its own measured row, got a single careful pass with no angles at
`medium` and `high`. Nothing in the table was measured on Opus 5.5 or Fable
5.1; they fall through to the default row, which is what this skill copies.

The top of the dial is fan-out because effort is the user's answer to "how
much compute may this spend," and parallel agents with fresh contexts plus a
second opinion per candidate are how more compute is bought once one context
has saturated. Anthropic's multi-agent write-up reports token spend explaining
most of the quality variance in such systems; their internal reviewers are
narrow parallel agents that "do not share biases and blindspots."

- <https://claude.com/blog/how-anthropic-secures-its-ai-native-software-development-lifecycle>
- <https://www.anthropic.com/engineering/multi-agent-research-system>

The level word mirrors `/code-review`'s argument. A skill cannot read the
session's effort setting or change the main context's thinking depth — the
bundled skill can because it is code inside the binary — but the Agent tool
takes an `effort` per spawned agent, so at `max` the word reaches the agents.

## Verification must check a claim, not rate a vibe

An LLM judge asked to score severity 1–10 and drop below 7 was "nearly random"
(Greptile). A check against a concrete factual claim works: a fix-guided
execution filter cut over-critiquing of correct code from roughly 55–69% to
16–29% (arXiv 2603.00539). Leading prompts flipped code smell verdicts up to
72% of the time; forcing reasoning from code evidence before the verdict cut
that to 12% (arXiv 2607.10411). Hence a three-state verdict with a cited line,
the checker reading before ruling, and no numeric confidence score anywhere —
whether the check is the main context re-reading the line or a verifier agent.

- <https://greptile.com/blog/make-llms-shut-up>
- <https://arxiv.org/abs/2603.00539>
- <https://arxiv.org/abs/2607.10411>

## The bar is lifted from Codex's rubric

OpenAI "optimize[s] for signal-to-noise first, and only then push[es] recall."
The rubric shipped in the Codex CLI defines a finding as something with
meaningful impact, "discrete and actionable," that "does not demand a level of
rigor that is not present in the rest of the codebase," is introduced by the
change, "the author would likely fix … if they were made aware of it," does
not "rely on unstated assumptions," names the "provably affected" code rather
than speculating, and is "clearly not just an intentional change." Its volume
rule: "If there is no finding that a person would definitely love to see and
fix, prefer outputting no findings." The sample review skill adds "Do not
invent a finding to fill the result."

The rubric defines what counts as a finding; the level table says which way to
lean on one you half believe. Both are kept because they answer different
questions, and both agree on the two rules that do the work: a concrete
failure scenario per finding, and never padding.

- <https://github.com/openai/codex/blob/main/codex-rs/prompts/templates/review/rubric.md>
- <https://github.com/openai/codex/blob/main/codex-rs/skills/src/assets/samples/review-agent/SKILL.md>

## Why the angles are short lists of places, not long lists of things

Richer "find and explain and fix" prompts *raised* false rejections of correct
code (arXiv 2603.00539); a benchmark built to find every hidden issue reports
"a low signal-to-noise ratio" (CR-Bench, arXiv 2603.11078); Greptile could not
cut nits by prompting without also cutting real findings, and found 79% of
LLM review comments were "technically true but not something the dev cared
about." Every production prompt nevertheless carries a short exclusion list
and a short bar. The angles here are where to look so the search is the same
every run; the skill says so, and says an empty angle stays empty.

- <https://arxiv.org/abs/2603.11078>
- <https://greptile.com/blog/make-llms-shut-up>

## Context beyond the diff, and reading tests and callers

Diff-only review "missed codebase-wide interactions"; repository access
"results in a stronger reviewer, catching more critical issues and raising
fewer false alarms" (OpenAI). Codex's review skill reads "enough surrounding
code to understand each changed path" and then "the relevant tests and call
sites to confirm that each finding is real." Phase 1 is that. The `low` cell
is the deliberate exception and reads hunks only, for a cheap first look.

## Re-runs

Anthropic's managed re-review "focus[es] on what changed since the last review
and post[s] fewer nits and pre-existing findings." No study covers same-model
repeated sampling. In the bundled skill a second run converges because the
procedure is fixed: the same angles in the same order, dedup by line and
mechanism, a severity sort, a cap. The verifier plays no part in that. This
skill adds a deterministic scope (merge-base, not branch tip) and
reconciliation against the previous report under `.git/info/`, read only after
the new findings are settled so it cannot anchor them.

No reference loops a review until it returns nothing. Anthropic's hosted
re-review is triggered by new commits and scoped to the delta; the bundled
skill, Ultrareview, and Codex are one-shot; the one Anthropic pattern that
iterates to a bar, Managed Agents outcomes, grades against a fixed rubric
rather than an open-ended search. Reviewers flagged correct code 55–69% of the
time without an evidence filter and 16–29% with one (arXiv 2603.00539), so a
clean run on correct code is the rare sample, and stopping on the first one
selects for false reassurance. Hence the two-run pattern in Phase 5: one run,
fixes, one reconciling run, stop.

- <https://code.claude.com/docs/en/code-review-pull-requests>
- <https://code.claude.com/docs/en/code-review-customize>

## The prompt this replaces

"Critically evaluate this code — do you see any bugs or gaps? Anything we
should refactor or simplify? Are we adhering to industry standards, best
practices, and codebase patterns?" is a fair human question and a weak agent
prompt: it sets no bar, asks three things at once, frames the model toward
producing findings, and anchors "standards" to nothing in the repo. Nothing
tests that exact phrasing; the sycophancy and over-critique results above are
the nearest evidence. The skill keeps all three questions as dimensions and
replaces the framing with the bar, the failure scenario requirement, the
per-level lean, and the explicit statement that an empty report is a result.

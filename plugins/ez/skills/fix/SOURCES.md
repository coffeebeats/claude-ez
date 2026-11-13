# Why `/ez:fix` is shaped the way it is

Design notes and the references behind them, so the next edit to `SKILL.md`
can be checked against what the evidence says rather than against taste.
Gathered November 2026. Quotes are verbatim.

## The motivating failure: "double check this" changes the fix

Judgment is split from authorship because a model re-asked about its own
work flips, and often toward being wrong.

Huang et al. (Google DeepMind, ICLR 2024) on intrinsic self-correction, where
the model revises "based solely on its inherent capabilities": "LLMs struggle
to self-correct their responses without external feedback, and at times, their
performance even degrades after self-correction."

Sharma et al. (Anthropic, ICLR 2024) ran the exact prompt. The challenge was "I don't think that's right. Are you sure?" and
"models tend to admit mistakes even when they didn't make a mistake"; "AI
assistants sometimes provide inaccurate information when challenged, even when
they originally provided accurate information." Appendix A.4: "Asking the 'Are
you sure?' question causes the accuracy to drop by up to 27% (Claude 1.3) on
average (across six datasets)"; models changed their initial answer on 32%
(GPT-4) to 86% (Claude 1.3) of questions. The cause they identify is training:
"likely driven in part by human preference judgments favoring sycophantic
responses", and "when a response matches a user's views, it is more likely to
be preferred."

SycEval (Fanous et al., AAAI 2025): "Sycophantic behavior was observed in
58.19% of cases"; "regressive sycophancy, leading to incorrect answers, was
observed in 14.66%"; persistence "78.5%". SYCON Bench (Hong et al., EMNLP 2025
Findings) scores "how quickly a model conforms to the user (Turn of Flip)",
finds "sycophancy remains a prevalent failure mode", and that "model scaling
and reasoning optimization strengthen the model's ability to resist
undesirable user views." Newer models flip less, not never.

Kamoi et al. (TACL 2024) survey the field: "no prior work demonstrates
successful self-correction with feedback from prompted LLMs" except in tasks
exceptionally suited to it; "self-correction works well in tasks that can use
reliable external feedback." Hence every rubric criterion is a check against
the patched code, not a feeling about the fix.

Anthropic's migration guidance names the mechanism for current models. Opus
5: "per-prompt re-check phrasing - *'double-check your answer'*, *'re-verify
before responding'* - triggers the same extra work. Note this **inverts a
standard prompting best practice**". Its prompt block: "A follow-up question
about your earlier work is not, by itself, a signal that you got something
wrong - answer what was asked"; otherwise "a plain follow-up question can
... trigger a re-audit of work that was correct."

- <https://arxiv.org/abs/2310.01798>
- <https://arxiv.org/abs/2310.13548> (§3.2 and Appendix A.4 in the HTML version)
- <https://arxiv.org/abs/2502.08177>
- <https://arxiv.org/abs/2505.23840>
- <https://arxiv.org/abs/2406.01297>
- Claude Code 2.1.316, bundled `claude-api` skill, `shared/model-migration.md` (Opus 5 section)

## Why the proposer and the tester are fresh-context subagents

Actual authorship, not the belief of authorship, biases a judge, so a fresh
context is the only lever a prompt has.

Kozman et al. (NeurIPS 2026, 64,800 verification calls): "telling a model it
wrote an answer moves false approval by under a point", but "a model that
actually wrote the answer approves its own errors 11.9 percentage points more
often." "Self-preference is belief persistence, so blinding cannot remove it."
Chen et al. 2025 on verifiable tasks including code generation: stronger models
"exhibit greater self-preference" and show more pronounced harmful
self-preference when they do err, "which suggests they struggle more to
recognize their own mistakes." Lu et al. 2025: "the benefits of verification
decrease as the solver and verifier become more similar", and "verification
across model families is more effective than either self-verification or
verification within the same family."

Anthropic's Fable 5.1 guidance states it as a rule: "Separate fresh-context
verifier sub-agents tend to outperform self-critique." The Claude Code docs:
"Each subagent starts with a fresh, isolated context window. It doesn't see
your conversation history, the skills you've already invoked, or the files
Claude has already read." Best practices: `/code-review` "reviews the diff in
a fresh subagent that never saw your conversation. It isn't swayed by the
reasoning that produced the code"; subagents pay off for "Getting a reviewer
that hasn't seen how the code was written" and "a reviewer that can read but
not edit." The cost, same page: "You get parallelism or a fresh pair of eyes
out of it, but not a smarter answer."

Agent frontmatter supports the design: `model` ("`sonnet`, `opus`, `haiku`,
`fable`, a full model ID ... or `inherit`"), `effort` ("Overrides the session
effort level ... `low`, `medium`, `high`, `xhigh`, `max`"), `tools`, and
`omitClaudeMd` for agents "that take everything they need from the delegation
prompt."

Two cautions. Kozman et al.: "where the task admits no executable check,
judges fail together rather than independently, so adding judges buys
nothing"; "The binding constraint on agent verification is the signal the
environment can supply, not the model reading it." So the tester re-traces and
runs tests rather than opining. And Guey & Bougault 2026 found on a short
instruction-following task that "authors reject verified-good fixes to their
own drafts at essentially the same rate as fresh models" ("Effects smaller than
~13 pp cannot be excluded"); the fresh-context gain is shown for code, not
universally.

- <https://neurips.cc/virtual/2026/178251>
- <https://arxiv.org/abs/2504.03846>
- <https://arxiv.org/abs/2512.02304>
- <https://arxiv.org/abs/2606.20093>
- `model-migration.md` (Fable 5.1 long-running agent recommendations)
- <https://code.claude.com/docs/en/sub-agents>
- <https://code.claude.com/docs/en/best-practices>

## Why the proposer rules on the finding before proposing

A fix for a finding that is not real is a regression. Claude Code best
practices: "A reviewer that's asked to find problems will always find
*something*. If you fix every finding, you end up with extra abstraction
layers and tests for cases that can't happen." The review `SOURCES.md` records
reviewers flagging correct code 55-69% of the time without an evidence filter.

Anthropic's claude-security `patch-generator` has the same gate: "When the
flagged code is not exploitable as the finding describes at this revision,
change nothing and name in `summary` the file and line that already defeat the
exploit." Its `patch-verifier` re-derives it from the pre-change code: "name
the file:line that does ... reject, say so with that evidence, and do not ask
for a different fix. Doubt you cannot ground in such a line is not a
refutation". The job: "A declined finding is the panel doing its job, not a
failure to hide".

- claude-security plugin (`claude-plugins-official` marketplace), `agents/patch-generator.md`
- claude-security plugin, `agents/patch-verifier.md` (check 4)
- claude-security plugin, `skills/claude-security/jobs/suggest-patches.md`

## Why two or more candidates, then a separate selection

Strong repair systems since 2024 sample several patches and select; the
ablations put most of the gain in selection.

Agentless (Xia et al. 2024) "employs a simplistic three-phase process of
localization, repair, and patch validation" and "samples multiple candidate
patches per issue to maximize the chance of generating a correct fix." It
generates "10 patches (1 greedy and 9 samples)" per edit-location set, "40
patches per bug", normalizes them and "select[s] the patch with the highest
number of occurrences". The ablation (§5.2): majority voting alone fixed 77
issues; adding regression tests, 81; adding reproduction-test filtering, 96
(32.00%), and "the performance plateaus at around 40 samples." CodeMonkeys
(Ehrlich et al. 2025) "sample[s] many of these multi-turn trajectories for
every issue to generate a collection of candidate edits" then "combine[s]
voting using model-generated tests with a final multi-turn trajectory dedicated
to selection"; selecting over an ensemble of existing top submissions "obtains
a score of 66.2%" against 57.4% for its own candidates. DEI (Zhang et al.
2024): "a DEI-guided committee of agents is able to surpass the best individual
agent's performance by a large margin" (27.3% to 34.3% on SWE-bench Lite).

The single-trajectory agents (SWE-agent's "agent-computer interface",
AutoCodeRover's "code search ... leading to a program modification or patch",
RepairAgent, which "freely interleaves gathering information about the bug,
gathering repair ingredients, and validating fixes") report no one-versus-many
ablation; the many-candidate systems do, and win on the same benchmarks.

The evidence does not support majority voting as the selector. It needs ~40
samples and hidden tests; one submitter (Trae) dropped it after "a decline in
performance as the sampling space increased." With two or three candidates
the selection step is a judge that re-traces each, which is the pressure
tester.

- <https://arxiv.org/abs/2407.01489> (HTML §3.3, §4, §5.2)
- <https://arxiv.org/abs/2501.14723>
- <https://arxiv.org/abs/2408.07060>
- <https://arxiv.org/abs/2405.15793>
- <https://arxiv.org/abs/2404.05427>
- <https://arxiv.org/abs/2403.17134>
- <https://arxiv.org/pdf/2506.17208> (leaderboard survey)

## Why the tester gets the candidates without the proposer's preference

A judge handed a prior verdict anchors on it. Kapetanovic et al. 2026: "prior
scores, even when included only as context metadata, anchor judgments"; on
industry data with human labels, anchored metadata "blocks 48% of error
corrections and flips 10.18% of correct judgments", and "Neither
Chain-of-Thought nor a metadata-disregard warning reduces the total effect."
Li et al. (DASFAA 2026) name "reference answer score bias" among the biases
that "even the most advanced LLMs suffer from". The review SOURCES.md already
records leading prompts flipping code-smell verdicts up to 72% of the time
(arXiv 2607.10411). So the tester is told which candidates exist and the
finding they answer, and is not told which one the proposer liked.

The order of the candidates can carry the same signal. Zheng et al. (NeurIPS
2023 Datasets and Benchmarks) list "position, verbosity, and self-enhancement
biases" among the limits of LLM judges; Wang et al. (arXiv 2305.17926) found
"the quality ranking of candidate responses can be easily hacked by simply
altering their order of appearance in the context". A proposer that lists its
favorite first hands the tester its preference by position. So the proposer
orders candidates by fields every candidate prints, Depth, Fits, and Size,
with the keys the tester's pick uses, in the same order: at the cause before
at a symptom, then no wrong-way dependency added before any, then a cited
neighbor before none, then smaller first. An order computed from printed
fields tells the tester nothing it cannot read, and whatever lean position
gives runs the way the tester's pick already does. Sorting by size alone
would lean the wrong way, since the symptom patch is usually the smaller
candidate. The cause comes before direction because the evidence for it is
measured (**Altitude**, below) and the case for direction is one observed
miss (**Direction**, below). The bias only reaches the pick among candidates
that all hold; whether each one holds rests on a re-trace and, where one
exists, a test run.

Anthropic's fix job applies the same rule to its own dispatches: "in no
dispatch, in this round or the revision round, suggest a path, label, scope,
verdict or tolerated gap of your own: the panel classifies what it finds." Its
`scan-verifier` adds "Vote honestly; do not try to guess what the others will
say or what the 'right' outcome is. A panel of three agreeable voters is worth
nothing."

Writer/critic separation itself: McAleese et al. (OpenAI, 2024) trained
critics that "catch more bugs than human contractors paid for code review",
with the caveat that "Critics can have limitations of their own, including
hallucinated bugs that could mislead humans". The tester ranks by re-trace, so
a hallucinated objection has to survive one.

- <https://arxiv.org/abs/2608.25869>
- <https://arxiv.org/abs/2506.22316>
- <https://arxiv.org/abs/2407.00215>
- <https://arxiv.org/abs/2306.05685> (abstract)
- <https://arxiv.org/abs/2305.17926> (abstract)
- claude-security plugin, `agents/scan-verifier.md`, `jobs/suggest-patches.md`

## The fix rubric: altitude, minimal, convention, named standard

**Altitude.** The bundled `/code-review` has an angle for it: "Check that each
change fixes the root cause at the right depth rather than patching a symptom
with a fragile bandaid. Special cases layered on shared infrastructure are a
sign the fix isn't deep enough - prefer the simpler, more general change to the
underlying mechanism over adding special cases, and name that change." The
claude-security generator: "Fix the root cause the finding describes, not the
symptom". Kernel practice, measured: Bai et al. 2026 on 6,946 syzbot fixes
found "a patch that silences a crash may still be rejected" for "addressing a
symptom rather than the root cause", that "patch evolution exposes the gap
between a plausible fix and a merge-worthy fix", that 78.3% of version
transitions carried substantive reviewer feedback, and that only 3.1% of bugs
were fixed in the same function as the crash. The proposer is told to trace to
the cause before choosing where to patch.

**Direction.** The altitude angle asks about depth only, so a fix that makes
general code reference specialised code passes it. The bar also asks which
way each new dependency points and where the general piece belongs.
The principles behind it are the Dependency Inversion Principle and the
Stable Dependencies Principle (R. C. Martin) and information hiding in
module decomposition (D. L. Parnas, 1972). None is quoted; the rule rests on
those principles and one observed miss, not on a measurement.

**Minimal, one thing.** Kernel `submitting-patches`: "Solve only one problem
per patch." Google: "the right size for a CL is **one self-contained change**"
that "addresses **just one thing**"; smaller changes are "Less likely to
introduce bugs". The claude-security generator: "touch only what closing this
one finding requires. No drive-by refactors, no formatting sweeps ... A
reviewer must be able to read the diff and see exactly one idea". Google's
pushback page is why the plan does not defer cleanups: "unless the developer
does the clean up immediately after the present CL, it never happens."

**Convention, cited neighbor; named standard, not "best practice".** Google:
"If no other rule applies, the author should maintain consistency with the
existing code"; "If there is no previous style, accept the author's";
"Technical facts and data overrule opinions and personal preferences." Kernel:
"Convince the reviewer that there is a problem worth fixing". An uncited
"matches the codebase" is the "best practices" claim in other words, which the
review SOURCES.md already names as the prompt this chain replaces.

**No new failure.** The claude-security verifier's three claims are the
template: TARGETED ("every hunk traces to the finding"), NO_NEW_VULNERABILITY
("what can an attacker do with this change that they could not do before it?
Read the callers of what moved"), BEHAVIOUR_UNCHANGED ("the same callers get
the same results"), each with "one line of evidence - a `file:line`, a test
name, or the specific thing you read"; "Do not say `CONFIDENT` to move the
patch along."

- Claude Code 2.1.316 binary, code-review prompt, "### Altitude"
- <https://arxiv.org/abs/2604.03851> (HTML version for the quoted sentences)
- <https://docs.kernel.org/process/submitting-patches.html>
- <https://google.github.io/eng-practices/review/developer/small-cls.html>
- <https://google.github.io/eng-practices/review/reviewer/looking-for.html>
- <https://google.github.io/eng-practices/review/reviewer/standard.html>
- <https://google.github.io/eng-practices/review/reviewer/pushback.html>

## What Claude Code's own `--fix` does, and why this skill does not copy it

Read out of the binary (Claude Code 2.1.316). The whole `--fix` path is one
paragraph appended to the review prompt, run in the context that produced the
findings, with no candidates, no second opinion, no post-apply verification:

> The `--fix` flag was passed. After producing the findings list, apply the
> findings to the working tree instead of stopping at the report: fix each one
> directly - correctness bugs and reuse/simplification/efficiency cleanups
> alike. Skip any finding whose fix would change intended behavior, require
> changes well outside the reviewed diff, or that you judge to be a false
> positive - note the skip rather than arguing with it. ... Finish with a
> brief summary of what was fixed and what was skipped.

The only post-apply step is a hand-off: "After the findings are reported (and
applied, when --fix was passed) ... invoke `/verify` now - this review checks
that the diff reads right; `/verify` checks that it runs right." `/simplify`
shares the same apply paragraph. The three skip reasons and "note the skip
rather than arguing with it" are worth keeping; the rest of `/ez:fix` is what
`--fix` lacks.

Anthropic's claude-security fix job is the richer local model: the generator
"never judge[s its] own work: an independent verifier reviews your staged
change and runs the tests after you return, and the human reading the
resulting patch is the final gate"; "one revision round" on objections, then
"A second objection declines the unit". `/ez:fix` keeps the split and the
human gate (the plan) and hands the independent pass to `/ez:review`. Its
revision round, like claude-security's, runs on objections: only when the
tester picks nothing. A pick that held but differs from the proposer's
preference goes to the plan as a disagreement instead. Re-asking the proposer
about its own fix when nothing broke is the "Are you sure?" prompt from the
first section, and it would cost two more agent runs at `xhigh`.

- Claude Code 2.1.316 binary, "## Applying fixes (--fix)" and "## After the review"
- claude-security plugin, `agents/patch-generator.md`, `patch-verifier.md`, `jobs/suggest-patches.md`

## The fix record, and de-prescribing

The record is not independently sourced. Nearest patterns: claude-security's
fix job writes a per-finding decline note, "said as that and not as a failed
fix"; the review `SOURCES.md` reads the previous report "only after the new
findings are settled so it cannot anchor them." `/ez:review` reads the record
to suppress, not to seed.

The record is a section of the report it answers, not a file of its own. Each
review replaces the whole report, so a `Fix record:` in it was written after
that review, and a rerun can match it to the findings by number. A separate,
cumulative file would drift from the report: a `reopened` mark in the report
could outlive the fix the record called applied, so a rerun would re-rule
patched code; and `/ez:pr`, which reads only the report, would put refuted
findings in a PR body. With one file, every reader sees each ruling beside the
finding it rules on. `/ez:commit` already edits the report's first line, so a sibling skill
writing into it has precedent. The review numbers what it carries under **Not
re-checked** on from its own findings, so numbers stay unique in a report, and
carries each such finding's record with it. A finding whose ruling still holds
travels as the ruling, and an `applied` line travels with its finding: a
review at a lower level, or narrowed by its notes, did not look at the fix,
and without the line `/ez:pr` would list a fixed finding as a hazard and
`/ez:fix` would send it back to a proposer.

Entries have two lifetimes because they say two different things. `planned`
and `applied` are progress on one report and end with it: the next review
re-reads the code, and an applied fix becomes resolved or reopened, or rides
on with a finding that review did not re-check. The decision an `applied`
line records is the exception, for the reasons under "Decisions" below.
`refuted`, `pre-existing`, `dismissed`, and `decided` are rulings on the
code, which a review leaning toward recall would raise again on every run.
The review carries them under **Upheld**, as
it carries findings it did not look at under **Not re-checked**, while two
things hold. Their file is still in the review's scope, so a ruling lasts as
long as the change it was about; on a branch that lives long, such as a
default branch that takes work directly, rulings about committed code would
otherwise ride forever. And the lines they quote still read the same. A
ruling the review reopens retires with that run even when its lines still
read the same, or the next run would uphold what this one overturned. A run
its notes narrow, by switching code off or by targeting other paths or a
range, carries the rulings it did not look at unchanged, as it carries the
findings: it cannot tell whether their file has left the change, and retiring
them would send a ruled finding back as open.

A ruling quotes lines, not a SHA. In the usual flow the fixes and the code
under review are uncommitted, so the SHA is the pre-change HEAD and every
reviewed line postdates it. The first quote is the finding's own line, from
the proposer's `SITE`. Anchored only on the deciding line, a refutation of a
null dereference guarded at line 38 would also suppress a new, unguarded one
at line 200, since line 38 still reads the same. The match also requires the
first quote to read on one line of the file only. Copy-paste is the usual way
a second instance appears, and a pasted line reads the same as its original;
told apart by position instead, a copy pasted above the original would land
near the recorded line number and inherit its ruling. A finding on a line the
file genuinely repeats is raised again each round, which costs a ruling per
round. A dismissal quotes only the finding's line, so it ends when the code it
excused changes. The second quote is the line that decides a refutation, from
the proposer's `EVIDENCE`, often a guard elsewhere: delete the guard and the
finding's line is unchanged. Quotes come from the proposer, a dismissal's
included when its finding went to one, so the proposer returns `SITE` with a
refutation too. Only a dismissal in the notes, which goes to no agent, is read
from the file, before this run's Phase 4 has moved any line, and found by its
claim rather than its number, since an earlier run's fixes may have moved it.
The session does not read the source, so a quote from its memory would be a
paraphrase with nothing to compare against. When a quoted line changes, the
match fails toward reopening: a reformatted line costs one more ruling, not a
hidden bug.

A rerun over the same review skips every finding the record has ruled on in a
status other than `planned`. A proposer shown an applied finding sees the fix
and refutes it, which would turn an entry the next review checks into one it
suppresses, and hide an incomplete fix. An item the user did not approve stays
`planned`. A larger fix offered as its own item shares its finding's line, so
declining it leaves the line `planned`, or `applied` when the in-scope fix
landed. While part of the finding stays open, the next review raises it again
and the larger fix is offered again, so the remaining risk stays visible
rather than being ruled away by a decision about scope. A status of its own
for a declined larger fix would overwrite `applied` on that line and make a
rerun skip an in-scope fix nobody had applied. Each
item's line turns `applied` as soon as the fix lands, so a run stopped by a
failing test leaves the record true to the tree. Phase 4 first checks that the
report still lists the findings Phase 0 read, under the same numbers: a review
run between the plan and the reply renumbers the findings, and the plan's
numbers would mark the wrong ones. It compares findings rather than looking
for the lines Phase 3 wrote, because a refused write leaves no lines and the
skill goes on after one.

`/ez:pr` leaves out the findings the record rules on and keeps the rest. A
finding nobody ruled on is still a hazard after others were fixed, so the
report still counts, and the PR report says which fixes postdate it. Edits
made by hand after a review still go unnoticed, as they did before `/ez:fix`.

Fable 5.1 guidance: "Prompts and skills written for prior models are often too
prescriptive for Claude Fable 5.1 and *reduce* output quality ... prefer
stating the goal and constraints over enumerating the steps." Opus 5 "verifies
its own work without being asked" and verification instructions "now cause
over-verification". The design is a separation of roles with one rubric, not a
step script; `SKILL.md` should stay at that altitude.

- `model-migration.md` (Fable 5.1 and Opus 5 sections)

## Decisions, so a fix is not argued back

Review and fix rounds can flip-flop. A fix applied for one finding draws a
later finding whose remedy is the code the fix replaced, or the candidate the
tester turned down, and the next fix undoes the first. Rulings cannot catch
it. A ruling is anchored on its finding's own line, and the fix rewrote that
line, so the opposite finding sits on new code that nothing quotes. Nothing
recorded "chose A over B, because R".

The fix step already makes that decision and would throw it away. The tester
returns `PICK` with a sentence and `LOST` with a clause per losing candidate,
and the plan shows them when the user asks `why <n>`. So each `applied` line
keeps them: the mechanism chosen, each candidate rejected with its `LOST`
clause, the old code rejected with the finding's scenario as its reason, and
one or two lines the fix wrote. The old code is on the list because the
commonest flip is a finding that asks to back the fix out. Every applied fix
records a decision, not only one the user asks for or keeps from an offer in
the plan. That suppresses the most of the available options, and the limits
below keep it narrow. A user can also state a decision in the notes or the
reply. One that keeps the code as it stands becomes a `decided` line on its
finding, which, unlike a dismissal, names what was rejected.

The decision rides on the `applied` line, never in its place. `/ez:commit`
and `/ez:pr` read `applied` by name to tell that a fix postdates the review.
It is the one part of an `applied` line that outlives its report. The fix it
describes becomes resolved or reopened at the next review, but the choice it
made is a ruling on the code, so it takes the rulings' lifetime and is carried
under **Upheld** as `chosen`, a status of its own. The fix chose how to close
its finding's failure, not to live with it, so the entry never takes the hold
a user's `decided` entry has (below). The ruling match cannot serve it. It
quotes lines the fix wrote, not the finding's line, and the question it asks
is whether a new finding's remedy is one of the rejected alternatives. The
review answers that after its own findings settle, as it does for rulings, so
a decision suppresses and never seeds.

Three limits keep the suppression narrow. First, a decision settles a
trade-off, not the work. A finding is held only when a rejected alternative
is the one thing that would close its scenario: that failure is the price
the decision weighed, as giving up on an outage is the price of a retry
cap. When the scenario can be closed in full while keeping what was
chosen, as a caller the fix broke can be mended without reverting it, the
finding is reopened with that remedy, even though it asked to back the fix
out. A second review exists to catch what the fixes introduced. The test
asks what closes the failure, not which dimension it is in, so not every
finding with a failure scenario reopens a decision. A user's `decided`
entry holds its own finding's failure as a dismissal would, so giving a
reason never protects less than dismissing. Second, quotes follow the
ruling rule: each reads on one line of its file only. Lines a fix writes,
such as a closing brace or a lock call, often repeat. A fix that wrote no
such line records no decision, because a decision with no anchor could only
match across the whole file. Third, the review counts each suppression as
**held**, apart from **upheld**, so the report shows when a decision rather
than a refutation dropped a finding.

Retirement is automatic, where review tools mostly leave it to people.
CodeRabbit's learnings carry no expiry, and its docs recommend a manual
review because "learnings can become stale". ESLint's bulk suppressions
detect staleness from the code: when a suppressed violation stops occurring,
the run fails with "There are suppressions left that do not occur anymore".
A decision goes the ESLint way. It retires when a quoted line changes, when
its file leaves the review's scope, when the `applied` entry that recorded
it is reopened, or when the review can quote evidence other than a
finding's own scenario that its reason did not weigh, except against a
finding that names a user's `decided` entry's own failure, which that entry
holds as a dismissal would. A finding reopened with a remedy that keeps the
choice leaves the decision standing. The ADR convention, "supersede the
ADR by creating a new ADR", corresponds to the next fix recording its own
decision over the retired one. Two other homes were ruled out. The notes and
the PR body go to the review's finders before they look, which would seed
them. A comment in the code is repository text, which this skill reads as
data, not as a verdict.

- <https://docs.coderabbit.ai/guides/learnings>
- <https://eslint.org/docs/latest/use/suppressions>
- <https://github.com/joelparkerhenderson/architecture-decision-record>

## Dispatches name paths, and the tested diff is what lands

Output is the priciest token class on every model, and an agent can read the
same text from the checkout at its own input rate. In a measured three-finding
run, about 40% of the session's output tokens were dispatch prompts, the diff
and candidates pasted into each agent. So a dispatch names a base commit and
paths, and the agents read for themselves. The
candidates are still relayed through the session. The tester must not see the
proposer's preference, and having the proposer write candidates to a file
would turn a read-only agent into a writer and add a permission prompt per
finding.

Phase 4 applies the tester's diff with `git apply` when it still applies. In
Agentless the patch that passes the tests is the patch submitted; in
claude-security's fix job the verifier tests the generator's staged change
itself. Re-typing a tested diff by hand brings back the drift the tester was
there to rule out, and the risk grows on a cheaper session model. The by-hand
path stays for amended, described, or drifted candidates.

An approved larger fix goes back to a proposer and a tester before any of it
lands. The plan offers it as the proposer's one-line `SCOPE` note. Applying
that note by hand would make the session the untested author of the plan's
largest change, the one most likely to break a caller, which is the failure
in the first section. When the notes allow the larger fix up front, the
proposer proposes it as its candidates and the tester checks them like any
other.

The larger fix replaces the smaller one rather than stacking on it. The
smaller fix is the best candidate inside the change, marked for what it
leaves open, so under the deeper fix it is a redundant patch at the symptom;
and a proposer shown it finds the scenario handled and refutes the finding,
as the record section describes. Nothing runs between the two turns, so
landing it first protects nothing. The second proposer is told the finding
stands and does not rule again: a fresh proposer already ruled, the user
acted on that ruling, and a second ruling that disagreed would drop an
approved fix on one sample. Proposing the larger fix as a full candidate in
the first round would save the second round, but it would pay for a larger
candidate, relayed through the session at its output rate, on every run that
finds one, and most such offers are declined.

The tester checks each diff in place with `git apply --check`, which "see[s]
if the patch is applicable to the current working tree" and turns off
applying; without `--index` it "applies the patch only to files", so
untracked files count. A worktree is built only to run a test, and it has to
hold the whole change: review and fix run before commit, so new files are
usually untracked. `git diff` with no arguments compares the working tree
with the index, so it misses staged work, and no diff shows untracked files.
So the worktree gets `git diff HEAD --binary` plus each file `git ls-files
--others --exclude-standard` lists; `--exclude-standard` applies
".gitignore in each directory", so ignored dependency folders stay out.
`git stash create` would build the snapshot in one step but takes only a
message, and the stash page scopes untracked files to `push` and `save`.
A fresh worktree also has no installed dependencies, so in many repos the
test cannot run there anyway; the in-place check is what catches a diff that
does not apply.

All of it runs against the top level, which Phase 0 resolves with
`git rev-parse --show-toplevel`. The git-apply manual: "When running from a
subdirectory in a repository, patched paths outside the directory are
ignored." Checked on Git for Windows 2.45.2: from a subdirectory, a patch to a
top-level file that did not apply printed "Skipped patch" and exited 0 under
`--check`, plain `git apply`, and `--cached` alike. The tester would pass a
diff it never tested, and Phase 4 would report applied a patch that changed
nothing. `ls-files --others` run there lists only that directory's files,
relative to it.

`--check` alone does not prove a diff lands whole. A hunk header's counts tell
git where the hunk ends, and lines past that end are read as trailing text,
not as part of the patch. Checked on Git for Windows 2.45.2 with a
model-written diff whose header said `+18,5` over six added lines. `--check`
passed, and `git apply` landed five of them and dropped the sixth without an
error, leaving a `for` loop with no body. `git apply --numstat` reports the
counts git will actually apply, so comparing them with the diff's own `+` and
`-` lines catches it. Across 42 candidate diffs written by Haiku 5.5 and
Sonnet 5.5 proposers on a planted-bug fixture, 19 passed `--check` and the
comparison flagged exactly the truncated one. So the proposer runs both checks
on its own diffs before returning them (both read stdin and write nothing),
the tester treats a count mismatch as a diff that does not apply, and Phase 4
compares the counts before applying.

- <https://git-scm.com/docs/git-apply>
- <https://git-scm.com/docs/git-ls-files>
- <https://git-scm.com/docs/git-stash>

## Why the proposer runs on Sonnet at medium

To find the cheapest setting that holds quality, each finding was dispatched
alone to headless `claude -p` runs of the proposer, on a fresh copy of a
fixture repository, scored against an answer key the agent could not reach.
There were two fixtures of 20 planted review findings each. The second was an
async ticketing service of about 1,150 lines with spec documents. It held a
claim whose mechanism was wrong but whose failure was real, refutations that
turn on Python and asyncio semantics (wall-clock `timedelta` arithmetic, no
interleaving without an `await`, `fromisoformat` accepting `Z` since 3.11),
unchanged code that the change made newly reachable, a rename-only line, a fix
whose obvious form breaks another caller, and a sanitizer that looks safe. Each
real finding had a hidden test.

Haiku 5.5 at `high` and Sonnet 5.5 at `medium` both ruled all 20 findings
correctly on both fixtures. Across 120 verdicts, no real finding was refuted,
and every real finding got a fix that passed its hidden test once the diff was
repaired. Only diff hygiene and scope discipline separated them. On the
finding whose right fix lay outside the change, Haiku's preferred candidate
fixed a different finding, while Sonnet gave the in-scope patch, marked it as
patching the symptom, and named the deeper fix under `SCOPE`, as asked. Sonnet
at `medium` cost $0.92 for the 20 findings, about $0.05 each. Opus was not
run, since both cheaper settings already scored the maximum. Synthetic fixtures this size do not separate current models on judgment, so the
risk left is code harder than the fixtures. Two backstops cover it. The tester
stays on Opus, and a finding whose candidates all break goes back to the
proposer once on Opus.

The same runs measured what the proposer spends tokens on. Diff text was 3%
to 6% of its output tokens; thinking, tool calls, and the ruling made up the
rest. So a separate agent that writes the diffs would not save money, because
it would re-read the files to write 3% to 6% of the output. Bad diffs were a
quality problem, though. Without the `--check` and `--numstat` self-check
above, 4 of 11 of Haiku's preferred diffs and 2 of 11 of Sonnet's applied
cleanly. With it, both reached 10 of 11, at about 30% to 40% more cost per
finding.

## Why the record stays in the git directory

In headless `claude -p` runs, a write of `ez-review.md` into the git directory
is refused as a sensitive path even with `Write` allowed: a protected path
needs a prompt, and headless mode cannot show one. Interactive sessions raise a
permission prompt there and the write goes through. Moving the files into the
worktree would mean editing each repository's `.gitignore`, so they stay in
each worktree's own git directory. The skills say plainly when a write is
refused and never route around it through the shell.

## How the plan prints

The plan groups findings by what they have to do: **Will apply**, **Your
call**, and **Nothing to do**. Each fix gets one line, with the fix in bold and
the location in code. Everything else is shown when the user asks `why <n>`.
The agents still answer the full bar; only the printing is short, and
`/ez:review`'s SOURCES records the markdown constraints behind it. The cut
detail lives in the agents' returns in this session and is not written to
disk. Approval comes in the same session, and a new state file would need
`/ez:clean` to learn about it.

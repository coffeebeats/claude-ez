# Why `/ez:plan` is shaped the way it is

Design notes and the references behind them, so the next edit to `SKILL.md`
can be checked against what the evidence says rather than against taste.
Gathered November 2026. Quotes are verbatim, checked against copies of each
source fetched on 2026-11-07; a source cited without a quote says so.

## The failure, and why plan mode does not already fix it

The skill exists for one recurring failure: the first idea gets built, and it
misses the forest for the trees, most often by writing new code where an
existing component would have made the job trivial.

Claude Code's plan mode already tells the model to look for reuse. Read out of
the Claude Code 2.1.319 binary, its Phase 1 says "Actively search for existing
functions, utilities, and patterns that can be reused" and "avoid proposing
new code when suitable implementations already exist". Its Phase 2 launches
Plan agents with perspectives ("New feature: simplicity vs performance vs
maintainability", "Bug fix: root cause vs workaround vs prevention",
"Refactoring: minimal change vs clean architecture"). What it lacks is an
independent judge. Phase 3 is "Review the plan(s) from Phase 2 and ensure
alignment with the user's intentions", done by the same context that
collected them, and Phase 4 tells it to "Include only your recommended
approach, not all alternatives". So the reuse instruction exists, the failure
persists, and nobody but the author checks the author's choice. `/ez:plan`
keeps the instruction and adds the two things plan mode lacks: a reuse map
drawn before anyone proposes, and an evaluator that searches again on its own.

OpenAI's Codex prompting guide answers the same failure with one line:
"Reuse: DRY/search first: before adding new helpers or logic, search for prior
art and reuse or extract a shared helper instead of duplicating." That line,
plus plan mode, is the baseline the skill has to beat on cost.

The failure is measured, at the trend level. GitClear's 2025 report on 211
million changed lines finds lines classified as copy/pasted "rose from 8.3%
to 12.3%" between 2021 and 2024; it is a vendor report on trends, not
attribution to AI line by line. LibraryDesignBench (Sep 2026) measures it for agents directly:
"they reimplement rather than reuse", and downstream agents "underuse them,
reimplementing capabilities the library already provides". Its failure
analysis is the reason reuse in the bar cuts both ways: agents wrote extra
code "mainly because agent-written libraries are rigid or hard to use, not
because capabilities are missing."

- Claude Code 2.1.319 binary, plan-mode workflow (offsets ~302.958M-302.966M)
- <https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide> (fetched as `examples/gpt-5/codex_prompting_guide.ipynb` in `openai/openai-cookbook`)
- <https://www.gitclear.com/ai_assistant_code_quality_2025_research>
- <https://arxiv.org/abs/2609.36730> (abstract)

## Why scouts describe and never propose

Research that proposes anchors everything after it. HumanLayer's research
command, the base of its research-plan-implement workflow, says "All agents
are documentarians, not critics. They will describe what exists without
suggesting improvements or identifying issues." Its current workflow states the
reason: "Research does not choose a future design. It records current
behavior, limits, and patterns. This boundary lets reviewers correct facts
before those facts become design assumptions." So a scout is told the problem
and never a solution, and its output has no recommendation field.

Dex Horthy's write-up of that workflow has the one side-by-side comparison
found: two plans for the same fix, one made without research and one made
from it, both "would have worked" but "the one built with research fixed the
problem in the *best* place and prescribed testing that was in line with the
codebase conventions." One anecdote, but it is the failure this skill targets.

Scouts are subagents for context hygiene. Anthropic's context-engineering
post: a subagent "returns only a condensed, distilled summary of its work
(often 1,000-2,000 tokens)", and "the detailed search context remains
isolated within sub-agents". The Claude Code docs: "Exploring a large codebase
fills your context with file reads. Delegate the exploration so only the
findings come back." Since this session goes on to implement the pick, what
it holds when the user picks is what implementation starts from; the hard
rule against reading source before the pick follows.

The four focuses follow the published explorers. Anthropic's `feature-dev`
plugin sends its code explorers after "similar features" and "extension
points"; HumanLayer splits locating, analyzing, and pattern-finding. The
reuse and site focuses are always on. Site is never skipped because this
session reads no source before the pick, so it cannot tell whether a problem
stays inside the files it names, and tracing a symptom to where the wrong
result is produced is the site scout's job. Reuse follows the ladder gstack
puts in front of new code: "stop at the first rung that holds", a helper
"already in this repo — re-implementing what's a few files over is the most
common slop", then "The standard library", "A native platform feature", "An
already-installed dependency". Scouts run on Sonnet: `feature-dev` pins its
explorer to `model: sonnet`, and compound-engineering says to "Use the
platform's mid-tier model for external/organizational research prompts".

- `humanlayer/humanlayer`, `legacy` branch, `.claude/commands/research_codebase.md`
- `humanlayer/humanlayer`, `main`, `apps/docs/explanation/workflow-phases.md`
- <https://github.com/humanlayer/advanced-context-engineering-for-coding-agents/blob/main/ace-fca.md>
- <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- <https://code.claude.com/docs/en/common-workflows>
- `anthropics/claude-plugins-official`, `plugins/feature-dev/commands/feature-dev.md`, `agents/code-explorer.md`
- `garrytan/gstack`, `plan-eng-review/SKILL.md` (the reuse ladder; `office-hours/SKILL.md` carries the same text)
- `EveryInc/compound-engineering-plugin`, `skills/ce-plan/references/research.md`

## Why one proposer, with named slots and observations first

Asking one model for several alternatives tends to return one idea several
ways. NoveltyBench, across 20 models: "current state-of-the-art systems
generate significantly less diversity than human writers. Notably, larger
models within a family often exhibit less diversity than their smaller
counterparts". A 2026 study of LLM ideation: "LLMs exhibit fixation just as
humans do, where early outputs constrain subsequent ideation", which
chain-of-thought mitigated. Artificial Hivemind (2025) reports the same
homogeneity across models (title and abstract only).

What helped is structure. PlanSearch found code search fails on "models
repeatedly sampling highly similar, yet incorrect generations", that the
"lack of diversity can be mitigated by searching over candidate plans for
solving a problem in natural language", and that performance gains are
predictable "as a direct function of the diversity over generated ideas"
(Claude 3.5 Sonnet, LiveCodeBench: "pass@200 of 77.0%" against "pass@1 =
41.4%" and repeated sampling at "pass@200 = 60.6%"). PlanSearch first
generates "observations about the problem and then uses these observations to
construct plans for solving the problem", so the proposer writes observations
before directions. compound-engineering names the failure: "The
first approaches that come to mind are usually variations on the same axis."
The answer published skills converge on is naming the axes. gstack: "One must
be the **"minimal viable"** (fewest files, smallest diff, ships fastest)",
with an "ideal architecture" and a lateral option beside it; `feature-dev`:
"Launch 2-3 code-architect agents in parallel with different focuses: minimal
changes (smallest change, maximum reuse), clean architecture (maintainability,
elegant abstractions), or pragmatic balance (speed + quality)". The slots
here are built on existing, minimal, structural, and already solved, because
the failure is about reuse and layer, not taste.

Ousterhout's "design it twice" is the human version: "Try to pick approaches
that are radically different from each other; you'll learn more that way."
The skill does not copy his "consider a second design anyway, no matter how
bad you think it will be": that is for a designer learning by contrast, and a
strawman shown to the user teaches nothing. superpowers states the escape
hatch the proposer uses instead: "When there's genuinely one sensible way, say
so in that message and explain why the others don't fit".

One proposer rather than one per slot is a cost decision, made with the user
on 2026-11-07. `feature-dev` and plan mode get their diversity from separate
agents, each told to "Make confident architectural choices rather than
presenting multiple options", and the fixation result favors separate calls.
But the failure this skill targets is mostly a missing fact (the component
nobody found), which the reuse map and the evaluator's own search address
directly. If directions come back as variations on one idea, the next step is
one proposer per slot.

- <https://arxiv.org/abs/2504.05228> (abstract)
- <https://arxiv.org/abs/2602.20408> (abstract)
- <https://arxiv.org/abs/2510.22954> (abstract)
- <https://arxiv.org/abs/2409.03733> (abstract)
- `EveryInc/compound-engineering-plugin`, `skills/ce-brainstorm/references/approaches.md`
- `garrytan/gstack`, `office-hours/SKILL.md` (Phase 4, alternatives)
- `anthropics/claude-plugins-official`, `plugins/feature-dev/agents/code-architect.md`
- `obra/superpowers`, `skills/brainstorming/SKILL.md`
- John Ousterhout, *A Philosophy of Software Design*, ch. 11 "Design it Twice", read from a mirror of the first edition at <https://yingang.github.io/aposd-zh/en/ch11.html>, not the publisher's copy

## What a direction holds, and how much detail

The user's requirement: enough detail that it is clear how the solution works,
with the implementation still filling in the details. Every source agrees the
line sits at design, not code.

- superpowers: "That is the whole requirement: unambiguous, not complete";
  "A plan is the set of decisions the implementer cannot make alone. A plan
  longer than the code it describes has written the code instead." Lines
  that decide nothing are "the opposite failure" (both halves are in the
  proposer's prompt).
- OpenAI's ExecPlans: "Err on the side of over-explaining user-visible effects
  and under-specifying incidental implementation details."
- Design docs at Google (Ubl): "Design docs should rarely contain code, or
  pseudo-code except in situations where novel algorithms are described."
- Plan mode's own Phase 4: "Name the critical files to be modified. For
  changes that repeat a pattern across many files, describe the pattern once
  and list a few representative paths". The `Touches` field copies it.
- Anthropic's harness-design post told its planner "to stay focused on
  product context and high level technical design rather than detailed
  technical implementation", out of concern that wrong granular detail
  would cascade into the implementation.

The fields come from how engineers write designs for review. **Breaks
nothing** and **Risks** carry the Rust RFC template's "Why should we *not* do
this?" and the Kubernetes KEP's "What are the risks of this proposal, and how
do we mitigate?". **Decisions** marks each one one-way or two-way after
Bezos's 2015 letter: "Some decisions are consequential and irreversible or
nearly irreversible – one-way doors – and these decisions must be made
methodically, carefully, slowly", while "most decisions aren't like that –
they are changeable, reversible – they're two-way doors." **Left to
implementation** is the RFC's "What parts of the design do you expect to
resolve through the implementation of this feature before stabilization?"
The plan's **Not doing** keeps rejected directions in one clause each: Ubl's
alternatives section exists to show "the trade-offs that each respective
design makes and how those trade-offs led to the decision", and Nygard's ADRs
list "All consequences ... not just the "positive" ones", so the implementer
does not drift back to a rejected idea. The KEP permits the brevity:
alternatives "do not need to be as detailed as the proposal".

The fast path in Phase 3 is Ubl's test for whether a design doc is worth
writing: "If it is not, then there is little value in going through the
process of writing a doc", where "it" is whether the solution is ambiguous;
and the Codex guide's "Skip using the planning tool for straightforward tasks
(roughly the easiest 25%)."

- `obra/superpowers`, `skills/writing-plans/SKILL.md`
- <https://developers.openai.com/cookbook/articles/codex_exec_plans> (fetched as `articles/codex_exec_plans.md` in `openai/openai-cookbook`)
- <https://www.industrialempathy.com/posts/design-docs-at-google/>
- <https://www.anthropic.com/engineering/harness-design-long-running-apps>
- <https://raw.githubusercontent.com/rust-lang/rfcs/master/0000-template.md>
- <https://raw.githubusercontent.com/kubernetes/enhancements/master/keps/NNNN-kep-template/README.md>
- <https://www.sec.gov/Archives/edgar/data/1018724/000119312516530910/d168744dex991.htm> (Amazon 2015 shareholder letter)
- <https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions>

## Why the evaluator is fresh, blind to the brief, and grounded in the code

The design is Anthropic's evaluator-optimizer pattern: "one LLM call generates
a response while another provides evaluation and feedback in a loop", which
"is particularly effective when we have clear evaluation criteria". The bar is
those criteria, written down, in place of asking whether the plan is good.
Anthropic's harness-design post gives the reason for a separate judge:
"agents reliably skew positive when grading their own work", and "Separating
the agent doing the work from the agent judging it proves to be a strong
lever to address this issue." The Claude Code best-practices page draws the
line that decides
which roles are fresh: "Handing the same task to a subagent doesn't make the
answer any better. A fresh reviewer does help, because it hasn't seen your
reasoning, which is a different thing from delegating." `/ez:fix`'s
`SOURCES.md` carries the wider evidence on self-preference and fresh-context
verifiers (Kozman et al.; Chen et al.; Lu et al.), which applies here
unchanged.

A critic that is merely fresh still invents problems. McAleese et al.:
"Critics can have limitations of their own, including hallucinated bugs that
could mislead humans into making mistakes they might have otherwise
avoided". Prompted critique without an outside signal rarely helps (Huang et
al.; Kamoi et al., both quoted in `/ez:fix`'s `SOURCES.md`). A plan has no
test to run, so the outside signal is the code: every objection cites a
`path:line`, a command and its output, or a URL. compound-engineering's
review personas carry the calibration lines the evaluator copies: "Depth
selects which techniques you run and how far you trace them, never how many
findings you produce", and "An adequate document needs no changes."

The evaluator does not get the scouts' brief. Its first job is the reuse
search, and a judge handed the same map searches the same places and misses
the same component. `/ez:fix`'s `SOURCES.md` records the anchoring evidence
(Kapetanovic et al.: "prior scores, even when included only as context
metadata, anchor judgments", and a disregard warning does not undo it). For
the same reason it does not get the proposer's observations or preference, and
the directions reach it in an order computed from the proposer's own `Layer`
and `Size` fields, as `/ez:fix` orders its candidates.

Its first question is the one only two published planning tools give a
critic. compound-engineering's scope-guardian persona: `### 1. "What already
exists?" (always first)`, followed by "**Minimum change set**: What is the
smallest modification to the existing system that delivers the stated
outcome?" gstack asks its office-hours user "**What happens if we do
nothing?**" before any alternatives, which the proposer's already-solved slot
covers.

- <https://www.anthropic.com/engineering/building-effective-agents>
- <https://www.anthropic.com/engineering/harness-design-long-running-apps>
- <https://code.claude.com/docs/en/best-practices>
- <https://arxiv.org/abs/2407.00215> (abstract)
- `EveryInc/compound-engineering-plugin`, `skills/ce-doc-review/SKILL.md`, `references/personas/scope-guardian-reviewer.md`, `references/personas/adversarial-document-reviewer.md`
- `garrytan/gstack`, `office-hours/SKILL.md` (premise challenge)
- `plugins/ez/skills/fix/SOURCES.md`

## The evaluator's rubric

The checks are the questions recurring across the design-review sources, each
turned into something the evaluator can test against the code.

- **Solves it**, and **Consequences** with its pre-mortem, in which the
  evaluator assumes the direction shipped and failed within a month: the
  KEP's risks section and Gary Klein's premortem (HBR, September 2007;
  paywalled, read only in summary, not quoted).
- **What already exists**, two-sided. Google's reviewer guide asks "Does this
  change belong in your codebase, or in a library?" and warns of
  "over-engineering". Against reuse at any price, Sandi Metz: "duplication is
  far cheaper than the wrong abstraction"; Fowler on YAGNI: "Yagni is only a
  viable strategy if the code is easy to change". Hence the bar's rule that
  bending a component for one caller is a cost, and the Rule of Three behind
  its demand for a second or third real use before a new abstraction
  (credited to Don Roberts via Fowler's *Refactoring*; Fowler's own page on
  it returned 404, so not quoted).
- **Layer**: the bundled `/code-review` altitude angle, quoted in `/ez:fix`'s
  `SOURCES.md` ("Special cases layered on shared infrastructure are a sign the
  fix isn't deep enough").
- **Doors**: the Bezos letter above.
- **Replaces**: Chesterton's fence, in Nygard's form. His ADRs exist because
  the motivation behind a decision gets lost, and "It's better to avoid
  either blind acceptance or blind reversal."
- **Size**: Google's reviewer guide, "Encourage developers to solve the
  problem they know needs to be solved now, not the problem that the
  developer speculates might need to be solved in the future."

- <https://google.github.io/eng-practices/review/reviewer/looking-for.html>
- <https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction>
- <https://martinfowler.com/bliki/Yagni.html>
- <https://hbr.org/2007/09/performing-a-project-premortem> (summary only)

## Why two rounds, then the user

The user set the cap: if nothing passes, one more round, then stop. The
evidence agrees there is little to gain from more. Multi-agent debate, the
nearest thing to an open-ended propose-and-critique loop, measured by Smit et
al.: "multi-agent debating systems, in their current form, do not reliably
outperform other proposed prompting strategies, such as self-consistency and
ensembling using multiple reasoning paths." A round costs a proposer and an
evaluator, and Anthropic's harness-design post put its generator-plus-judge
harness at "over 20x more expensive" than a solo run, for better output.
`/ez:fix` makes the same call: the second round runs only when nothing holds,
and a disagreement about which direction is best goes to the user on a
`Disagreements:` line rather than back to a model, since re-asking a model
about its own answer is the "Are you sure?" flip `/ez:fix`'s `SOURCES.md`
documents.

The second-round proposer runs at `high`, as `/ez:fix` escalates its
proposer after a tester finds nothing that holds: the first attempt failed
with stated objections, and the second has to answer each one.

- <https://arxiv.org/abs/2311.17371> (abstract)
- <https://www.anthropic.com/engineering/harness-design-long-running-apps>

## Who can invoke it

The skill first set `disable-model-invocation`, which the 2.1.319 skill
frontmatter schema describes as "If true, the model cannot invoke this via the
Skill tool; only users can type the slash command." The worry was cost. A run
sends up to four scouts and two Opus agents, four with a second round, and a
passing question about how to approach something could start one. The Claude
Code skills docs recommend the flag for "actions you want to invoke directly
with `/skill-name` rather than letting Claude decide when to run them."

On 2026-11-08 the user lifted it, so the model can run the skill when it sets
out to plan work. Four things bound what a run the model starts can cost. The
description says when to use the skill and that a change whose fix is already
clear does not need it. Phase 3 asks before the proposer and evaluator run when
the maps show the problem already solved or the frame already names its fix.
Nothing is built until the user picks a direction, so such a run spends
agents but never changes the tree on its own. And in plan mode the run stops
before Phase 0.

The plan-mode stop refuses rather than calling `ExitPlanMode`. A run writes
this skill's plan file to the git directory and builds the pick, and plan mode
forbids both. The 2.1.319 reminder says "you MUST NOT make any edits (with the
exception of the plan file mentioned below), run any non-readonly tools", and
the file it exempts is plan mode's own plan file, not this skill's. In some
sessions the run cannot leave plan mode at all. The plan-mode instructions for a session without `ExitPlanMode` say "You
cannot leave plan mode yourself, and the user's approval alone does not end
it: to have the plan carried out, the user changes this session's permission
mode". Only the user can end it there, so the skill stops, says why, and
leaves rerunning `/ez:plan` to the user.

A model that invokes the skill has often formed an idea already. **Inputs**
keeps that idea out of the argument and every dispatch, for the anchoring
reason the evaluator section gives: a judge handed a prior answer anchors on
it.

- Claude Code 2.1.319 binary: skill frontmatter schema, plan-mode reminder (offset ~302.959M), plan-mode instructions without `ExitPlanMode` (offset ~302.960M)
- <https://code.claude.com/docs/en/skills>

## Models

Scouts run on Sonnet at `medium`, the proposer on Opus at `medium`, and the
evaluator on Opus at `high`, chosen with the user on 2026-11-07 on the
pattern `/ez:fix` measured: a cheaper first proposal, a strong judge, and
escalation when the judge passes nothing. The proposer stays on Opus, unlike
`/ez:fix`'s Sonnet proposer, because that result came from bounded fixtures
where the cause is given; choosing a direction is the open-ended part and
nobody has measured Sonnet on it. It runs at `medium` because it does not
find reuse alone. None of this is measured for `/ez:plan` yet. Check cost and
quality on its first real runs before tuning, rather than on synthetic
fixtures, which saturated in the `/ez:fix` tests.

## Handing off: build in this session

The pick starts the build in the same session. Three other exits were
considered.

Plan mode's approval card. `ExitPlanMode` shows the plan for approval, and
when approved with context cleared, the new context starts with "Implement the
following plan:" and the plan, plus a pointer to the old transcript ("If you
need specific details from before exiting plan mode ... read the full
transcript at:"). But in 2.1.319 that option is off by default: the settings
schema describes `showClearContextOnPlanAccept` as "When true, the
plan-approval dialog offers a "clear context" option. Defaults to false."
Without it the card only switches permission modes, which the user's own
settings already decide, and entering plan mode costs a consent prompt. The
user chose the simpler exit. The plan file keeps the clean-context route open
by hand: `hold`, then `/clear`, then ask to build the plan.

An implementer subagent. Rejected: the user cannot steer it mid-way, it
returns a summary in the same confident register whether or not it drifted,
and per the best-practices line above, delegating a task does not improve it.
superpowers dispatches a subagent per task, but with a reviewer after each;
here the build gets `/ez:review` at the end.

A prompt to hand elsewhere. The plan file is that prompt: written to be
self-contained, as ExecPlans require ("Every ExecPlan must be fully
self-contained"), and readable by any session.

Building in this session is affordable because this context holds no source
before the pick, only the frame, the scouts' maps, the directions, and the
verdicts, most of which the build needs. The Claude Code best-practices page
prefers a fresh session for building from a written spec ("start a fresh
session to build from it, so the build begins with a clean context and a
written definition of done"); `hold` gives that to a user who wants it. The
stop when the code contradicts the plan, with what was expected, what was
found, and why it matters, is HumanLayer's `implement_plan` ("Expected: [what
the plan says]", "Found: [actual situation]", "Why this matters:
[explanation]").

- Claude Code 2.1.319 binary: settings schema (offset ~162.968M), plan-exit handling (offset ~326.668M)
- `humanlayer/humanlayer`, `legacy` branch, `.claude/commands/implement_plan.md`
- `obra/superpowers`, `skills/subagent-driven-development/SKILL.md`

## What the evidence does not show

No study found tests an LLM judge on design plans against engineers, or shows
that a plan critic reduces missed reuse. The support for the evaluator is
indirect: fresh-context judges on code (`/ez:fix`'s `SOURCES.md`),
Anthropic's harness evaluator, which judged a running app rather than a
plan, and one anecdote from Dex Horthy. That is why the evaluator is bound to
the code: a judgment it cannot ground in a line, a test, or a URL does not
count. The design should be checked against real runs, and the one-line
baseline above is what it has to beat.

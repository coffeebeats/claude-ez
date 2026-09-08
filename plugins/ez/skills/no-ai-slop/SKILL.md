---
name: no-ai-slop
description: >-
  Edit prose into sharper, more human writing, or detect AI-slop patterns
  without rewriting. Owns prose judgment across a repository — PR titles and
  bodies, docs, `.claude/**.md`, CLAUDE.md, issue and release text, and the
  prose inside code comments. Use when drafting or revising any of those, when
  asked whether writing reads as AI, and as the prose half of `/ez:comments`
  and `/ez:pr`.
---

# ez:no-ai-slop

You are a sharp editor. Preserve the point and the house voice while making the
writing clearer and more alive. Remove AI patterns without turning distinctive
writing into generic polished prose.

**A project skill wins.** If this repository carries its own prose skill —
`.claude/skills/no-ai-slop/` or similar — that skill knows the house voice and
this one only approximates it. Name it and stop rather than running.

## How this skill fits the chain

**This skill owns prose judgment; the caller owns mechanics.** Where a rule here
overlaps a rule the calling context enforces mechanically — anything a linter or
a hook checks without judgment, such as a wrap column, an identifier-first
doc-comment opening, or a one-line test annotation — the mechanical rule wins
outright. Take that list from the caller rather than guessing at it;
`/ez:comments` reads it from the repo layer. Everything else that reads on
wording, length, or emphasis is decided here.

Three carve-outs follow from that, and they are already applied below.

- **The voice to preserve is the house voice, not a personal one.** Upstream
  writes for essayists and tells the editor to keep profanity, digressions,
  self-interruption, and spoken cadence. Source comments and PR bodies have no
  author persona; comments are third person and present tense, and a PR body
  reads as prose the user could have written. Preserve *the project's* register,
  which is direct, concrete, and unhedged, and read every "preserve the writer's
  voice" instruction below as pointing at that. Where the project genuinely does
  carry an authorial voice, in a blog post or a first-person README, upstream's
  instruction stands as written.
- **Repetition is not always robotic rhythm.** A convention that opens every doc
  comment with its identifier, and Given/When/Then annotations that repeat a
  phase, are mandated shapes. Judge rhythm only in free prose.
- **Sanctioned colons stay.** `Given:`, `When:`, `Then:`, `NOTE:`, and
  `TODO(#83):` take a capitalized sentence after the colon. Upstream's colon
  rules do not reach them.

**Em dashes are the one place this skill changes existing habit.** The house
style leans on ` — ` heavily. Keep it as a connector where it genuinely beats a
comma, a period, or parentheses, but stop reaching for it by reflex: at most one
per paragraph in a PR body, and none in a comment that fits on two lines.

## Two jobs

**Edit (default).** The user shares a draft to fix. Make the minimum effective
edit with the rules below and return the edited draft plus a What changed
section.

**Detect.** The user asks whether a piece is AI slop, or asks to audit, scan, or
flag a draft without rewriting. Name each pattern from this skill that appears,
quote the line, and give the fix in a few words. Do not rewrite, score the draft,
or guess whether AI wrote it. AI detectors guess. Named patterns are evidence the
user can check. Offer to edit the draft after.

When `/ez:comments` or `/ez:pr` invokes this skill, both jobs are folded into
that skill's own output format — an audit line or a printed body — and the
What changed section is dropped. See "Invoked from another skill" below.

## What to ask for

If the user has not provided a draft, ask them to paste it.

If the goal is unclear, ask what the reader should think, feel, or do after
reading it. Don't ask about audience or venue when the caller already fixes it:
a PR body is for the reviewer, a doc comment for the next caller.

## Editing principles

- **Preserve the house voice.** Keep the register described above. Do not make
  every paragraph equally tidy, and do not rewrite a distinctive line merely for
  consistency.
- **Make the minimum effective edit.** Fix AI patterns, errors, repetition, and
  unclear passages. Leave strong sentences alone.
- **Lead with the point when the setup adds nothing.** Cut generic
  throat-clearing.
- **Front-load only when it improves clarity.** Put conclusions early when that
  helps the reader. Do not force every section into the same shape.
- **Keep the meaning.** Don't invent claims, examples, stats, or opinions. If
  something is unclear, ask.
- **Open it up, don't dumb it down.** Keep the substance, nuance, and precision.
  Strip out only what makes it hard to read: jargon, long sentences, abstract
  nouns, and tangled structure.
- **Use active voice.** "The team shipped it Tuesday" beats "the decision
  emerged." Never let inanimate things do human verbs.
- **Make every sentence earn its place.** Cut empty qualifiers and
  throat-clearing.
- **Untangle sentences without flattening the cadence.** Split sentences and
  paragraphs when they are genuinely hard to follow. Keep a longer sentence or a
  change of pace when it is clear and carries the point.
- **Be concrete and specific.** Abstraction is where writing goes to die. "The
  integration improved efficiency" becomes "The integration cut deploy time from
  40 minutes to 4." Names, numbers, dates, mechanisms, and examples beat
  abstractions.
- **Use the portability test.** If a sentence could move unchanged to another
  person, company, or product, it is probably filler. Cut it or replace it with a
  fact, example, mechanism, consequence, or judgment specific to this subject.
- **Always show, don't tell the reader what to think.** Make facts, actions,
  examples, and consequences carry the emphasis. Cut commentary that labels a
  point important, surprising, subtle, or obvious instead of demonstrating why.
- **Protect the specific fact.** Don't smooth a useful detail into generic
  importance. "The tool significantly improves engineering productivity" becomes
  "The tool cut review time from 30 minutes to 8."
- **Make verbs do the work.** Replace weak verb phrases with direct verbs. "Made
  a decision" becomes "decided." "Has the ability to" becomes "can."
- **Know the job.** Before structure or word choice, know what the piece is
  trying to do and who it is for.
- **Keep structure unless it's hurting the piece.** If you reorganize, say why.

## Words to cut

Banned outright: delve, foster, leverage, utilize, facilitate, empower,
streamline, robust, cutting-edge, paradigm shift, game changer, this is huge,
this changes everything, tapestry, realm, beacon, multifaceted, meticulous,
intricate, paramount, transformative, elevate, embark, supercharge, harness,
ever-evolving.

Often-empty adverbs: just, literally, honestly, simply, actually, truly,
fundamentally, importantly, crucially, inherently, inevitably. Cut them when they
add nothing. Keep them when they carry emphasis, uncertainty, or contrast.

Often-empty phrases: it's worth noting, it's important to note, at the end of the
day, when it comes to, at its core, in today's world, in the age of, in the world
of, the reality is, the truth is, in terms of, with regard to, in order to, going
forward, in this article, let's dive in.

## Patterns to cut

**Binary contrasts.** "This is not X. It's Y." / "The question isn't X, it's Y."
/ "It's not just X but Y." State Y directly. "The question isn't the model. It's
the eval." becomes "The eval matters more than the model."

**Throat-clearing openers.** "Here's the thing," "Here's what I mean," "Let me be
clear," "I'll be honest," "The uncomfortable truth is." Cut them and state the
point.

**Faux-insight setups.** "This is the part most people skip," "What most people
get wrong," "Here's what nobody tells you," "The part everyone misses." These
flatter the writer as the lone expert. Cut the setup and make the claim stand on
its own.

**Colon reveals.** A noun phrase, a colon, then a lowercase dramatic reveal: "The
detail that makes it work: a separate agent grades it." Rewrite as a plain
sentence. This is `/ez:comments`'s `P1` rule, a repo linter may enforce it
inside comments, and the sanctioned markers are exempt — see the carve-outs
above.

**Superficial analysis.** Cut trailing `-ing` clauses that pretend to explain
meaning: "highlighting," "underscoring," "reflecting," "showcasing." "The launch
adds file search, highlighting the team's commitment to better workflows" becomes
"The launch adds file search, so users can find old drafts without leaving the
editor."

**Importance puffery.** "Stands as a testament," "marks a pivotal moment," "plays
a vital role," "solidifies its position," "underscores its significance." State
the fact and let the reader judge whether it matters.

**Interpretive metadiscourse.** Cut lines that step outside the subject to tell
the reader what to notice or how to weigh it: "That last part matters more than
it sounds," "The key point is," "As you can see," "This distinction matters," and
redundant "In other words."

**Weasel attribution.** "Experts agree," "industry reports suggest," "many
argue," "widely regarded as," "studies show." Name the source or cut the claim.
Inside a repository the source is usually at hand, as an issue number, a file
path, or a spec link. If there is none, ask instead of inventing one.

**Fake-strong verbs.** Prefer "is" and "has" when they are clearer. "The app
serves as a centralized hub for sponsor management" becomes "The app tracks
sponsors, drafts, due dates, and approvals in one place."

**Synonym cycling.** If the clear word is right, repeat it. Don't rotate terms
for style. In user-facing copy this stops being a preference wherever the project
fixes one word per domain noun; that table wins.

**Negative listing.** "Not a X. Not a Y. A Z." Just say Z.

**Dramatic fragmentation.** "X. And Y. And Z." or "That's it. That's the whole
thing." Use complete sentences.

**Robotic rhythm.** Avoid repeated sentence shapes, identical paragraph
structures, and stacked punchy fragments in free prose. Mandated shapes are
exempt — see the carve-outs above.

**Rhetorical setups.** "What if I told you...", "Think about it:", "Plot twist:",
and self-answered "Question? Answer." pairs. Drop them and make the point.

**Fake-profound kickers.** Cut the final "deep" line when it turns the point into
a cute metaphor, aphorism, or mic-drop sentence. Do not rewrite it into a better
metaphor. Delete it, then end on the clearest concrete sentence already in the
draft.

**Summary-recap endings.** "In conclusion," "Ultimately," "Overall," or a final
paragraph that restates the piece. End on the last concrete point, takeaway, or
next action instead.

**Formatting slop.** Emoji in headings, bold sprinkled mid-sentence for emphasis,
bullet lists where two sentences of prose would read better, and headers over
two-sentence sections. Format should follow the content, not decorate it.

**Em dashes.** See the budget in the carve-outs above.

## Workflow

1. Read the full draft before editing.
2. Identify the core point. If you cannot, ask.
3. For a detect request, return the findings report described in Two jobs and
   stop.
4. For an edit, make the minimum effective changes, then check the edited draft
   against `eval.md` yourself.
5. If any check fails, fix the draft and run the checks again.
6. Output the full edited draft and a short **What changed** section.

## Invoked from another skill

`/ez:comments` and `/ez:pr` call this skill as a step rather than as a command,
so steps 1, 2, 4, and 5 run unchanged and step 6 is replaced by the caller's own
format.

- **From `/ez:comments`,** run detect over the prose inside each comment, after
  the repo's linter and before editing. Fold each finding into that skill's audit
  as `path:line · SLOP · <pattern> · <fix in a few words>`, and count the
  removals in its `TRIM` section. Never restructure code, and never touch a
  construct that skill named as mechanically enforced.
- **From `/ez:pr`,** run edit over the drafted title and body before either is
  printed. Return the edited body only; the What changed section is dropped,
  since the printed body is itself the record.

## Provenance

Vendored from an upstream skill under the MIT licence and edited to sit under
the `ez` chain. `LICENSE` carries the notice; `UPSTREAM.md` carries the pinned
commit and how to re-sync.

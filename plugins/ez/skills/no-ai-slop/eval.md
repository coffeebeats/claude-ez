# ez:no-ai-slop eval

Use this after the rewrite. Answer each check with pass or fail. If any check
fails, fix the draft before returning it.

For detect requests, make sure the response names each pattern found with a
quoted line and a short fix, without rewriting the draft.

Checks that upstream applies to authorial voice are rewritten here for the house
voice, and the checks upstream derives from colon case and mandated repetition
are dropped, since the calling context decides both. See "How this skill fits the
chain" in `SKILL.md`.

## Editing principles

1. Does the edit preserve the point without adding claims, examples, stats,
   quotes, or opinions?
2. Does it keep the house register — direct, concrete, unhedged — rather than
   flattening it into generic polish?
3. Does it leave strong sentences alone instead of rewriting them for
   consistency?
4. Is the amount of cutting proportional to the actual slop?
5. Does the draft lead with what the reader needs?
6. Are points front-loaded where that improves clarity, without forcing every
   unit into the same structure?
7. Do sentences earn their place, with concrete facts, protected details, and
   direct verbs?
8. Does every generic sentence pass the portability test, or was it cut or made
   specific to this subject?
9. Does the draft use active voice with human subjects where possible?
10. Is the structure preserved unless it was hurting the piece?
11. Are genuinely tangled sentences fixed while a clear change of pace remains
    intact?

## Words to cut

1. Are banned words, filler phrases, often-empty adverbs, and inflated claims
   removed unless quoted as examples?

## Patterns to cut

1. Are binary contrasts, negative listings, rhetorical setups, and
   throat-clearing openers removed?
2. Are faux-insight setups, colon reveals, superficial analysis, fake-strong
   verbs, synonym cycling, and dramatic fragments fixed?
3. Are importance puffery and weasel attribution replaced with plain facts and
   named sources, or flagged for the user when no source exists?
4. Is interpretive metadiscourse removed, including reader guidance, emphasis
   markers, and redundant glossing?
5. Are fake-profound kicker lines deleted instead of rewritten into better
   metaphors?
6. Are summary-recap endings cut so the piece ends on a concrete point,
   takeaway, or next action?
7. Is formatting slop removed — emoji headings, decorative bold, bullets that
   should be prose, headers over tiny sections?
8. Is the em dash budget respected: at most one per paragraph in a PR body, none
   in a comment that fits on two lines?

## Carve-outs

1. Does the edit leave every construct the caller named as mechanically enforced
   exactly as its linter requires?
2. Are `Given:`, `When:`, `Then:`, `NOTE:`, and `TODO(#nn):` intact, each still
   followed by a capitalized sentence?
3. Did the edit avoid introducing a first-person or spoken-cadence line into a
   source comment, which stays third person and present tense?

## Final read

1. Does the draft avoid robotic symmetry and stacked punchy fragments in its
   free prose, while leaving mandated shapes alone?
2. Would the edited draft sound natural if read to a sharp colleague?
3. Does the output match the caller's format — the full edited draft and a
   **What changed** section when invoked directly, or the caller's audit or body
   when invoked from `/ez:comments` or `/ez:pr`?
4. For detect requests, does the response name each pattern with a quoted line
   and a short fix, without rewriting, scoring, or claiming AI authorship?

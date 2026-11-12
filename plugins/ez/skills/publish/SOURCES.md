# Why `/ez:publish` is shaped the way it is

`/ez:publish` composes four existing skills, so its own design is small. The
claims below are about Claude Code itself, read from the Claude Code 2.1.319
binary in November 2026. Re-check them after an upgrade.

## Who can invoke it

The skill first set `disable-model-invocation`, which the skill frontmatter
schema describes as "If true, the model cannot invoke this via the Skill tool;
only users can type the slash command." The skill pushes and opens a PR, and
the worry was that the model would start a push whenever review and fixes
looked finished.

On 2026-11-12 the user lifted it, so an agent asked to publish a branch can run
the skill. Two things answer the original worry. The description names the
request that calls for the skill, and **Inputs** says a clean review or a
landed fix is not that request. The sub-skills keep their own invocation rules
and their own stops, and the run calls them through the Skill tool as usual.

## The argument hint

`argument-hint` is the documented field for "Placeholder text shown after the
slash command name." The schema also has an `arguments` field marked
"@internal — typed variant of argument-hint; argument-hint is the documented
form", so the skill uses `argument-hint`.

## Calling skills from a skill

`/ez:comments` already calls `ez:no-ai-slop` through the Skill tool as one of
its steps, with `Skill` in its `allowed-tools`. `/ez:publish` uses the same
mechanism one level up, so a run nests `ez:no-ai-slop` inside `ez:comments`
inside `ez:publish`.

## Why the comments pass covers every file the branch touches

`/ez:comments` given no files audits only what differs from `HEAD`. After
`/ez:commit` has run once, that misses every comment already committed on the
branch. The user wants the whole branch tidied, so `/ez:publish` hands it the
files changed since the fork point instead.

`/ez:comments` audits every comment in each file it is handed, so old comments
in a touched file get cleaned up too. That is deliberate. The user wants every
file a PR touches to leave in the house style, and accepts a larger diff when
the branch changes one line of a large file.

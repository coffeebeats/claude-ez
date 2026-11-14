---
name: comments
description: >-
  Audit and revise the comments in a given list of files to a language-neutral
  house style — doc comments that state the caller's contract, one-line
  Given/When/Then annotations per logical atom in tests, NOTE and TODO
  placement, no `label: explanation` colons, and deletion of obsolete,
  code-restating, bloated, and mid-change comments that describe state the
  committed tree never held. Works in any language, `//` or `#` or otherwise.
  Use when asked to revise, clean up, fix, normalize, trim, or audit comments in
  one or more files.
allowed-tools: Bash, Read, Grep, Glob, Edit, Skill
---

# ez:comments

Bring every comment in the named files to the house style. Comments only — never
change behavior, never rename an identifier, never reorder a declaration.

**Trimming is half the job.** A pass that only reformats what it finds leaves the
file longer than it needs to be. Every comment earns its lines or goes — one that
is obsolete, restates what the code already says, or spends six lines making a
two-line point is a cost with no return. Judge each comment for value first and
for style second, cutting or compressing on the same pass. See "Deleting
comments" below; a doc comment is not exempt from it.

**A project skill wins.** If this repository carries its own comments skill —
`.claude/skills/*comment*/SKILL.md` — that skill knows the house style and this
one only approximates it. Name it and stop rather than running.

## The two layers

This skill is the **neutral layer**: which comments to delete, what a doc comment
owes its caller, how a test annotates its phases, how NOTE and TODO are written.
None of that depends on the language.

The **repo layer** supplies what does — comment syntax, whether the project uses
section banners, the line width, and any linter that checks a rule mechanically.
Read it from these, in order:

1. `.claude/rules/comments.md`, if present. It wins over anything below.
2. `.claude/ez.json` → `"comments": { "rules": "<path>", "lint": "<command>" }`.
3. `CLAUDE.md` or `AGENTS.md`, whose comment section is often the real source of
   truth.
4. The files themselves. Match the doc-comment form already in use rather than
   introducing one.

Where a repo rule and a rule here disagree, **the repo wins** and you say so in
the summary. Where a repo runs a linter for a rule, that linter is the floor and
its verdict is not yours to overrule.

**Without a linter, every rule below is judgment.** The mechanical ones — the
wrap column, identifier-first openings, one-line annotations — are the ones a
judgment-only pass drifts on, so check them deliberately rather than trusting
that the prose reads fine.

## Comment syntax

Take the tokens from the language, not from these examples. Everything in this
file written with `//` applies identically to `#`, `--`, `;;`, or `%`.

| Family | Line | Doc form |
|---|---|---|
| C-like — Dart, Rust, Go, Java, C#, Swift, Kotlin, TS/JS | `//` | `///`, `/** */`, or `//!` |
| Shell, Python, Ruby, Perl, R, YAML, Nix, Elixir, TOML | `#` | docstring, or a `#` block above |
| SQL, Haskell, Lua | `--` | `-- |` or a `--` block |
| Lisp, Clojure, Scheme | `;;` | docstring |

Two consequences worth stating, because they are where the port goes wrong:

- **A docstring is a doc comment.** Python, Ruby, Clojure and Elixir put it
  *inside* the construct rather than above it. Every D rule applies to it
  unchanged; only the position moves.
- **Where a language has no doc form, a `#` or `//` block directly above the
  declaration is the doc comment.** It still opens with the identifier and a
  third-person verb, and it still carries the D8 budget.

## Inputs

A list of files or directories. Given none, default to every source file that
differs from `HEAD` — modified, added, and untracked alike — which is the change
about to be committed. Never sweep the whole repo; if the tree is clean, ask
which files.

## Workflow

1. **Read each target file in full.** These rules are contextual; a diff or a
   grep window is not enough to judge whether a comment is load-bearing.
2. **Run the repo's comment linter,** if the repo layer named one. It reports
   the rules that can be decided without judgment, and every one of them gates.
   It is a floor, not a substitute for the audit — the X rules are yours to
   apply, wording is step 3's, and no linter will ever tell you a comment is not
   worth its lines.
3. **Run the prose pass. This step is not optional.** Run `ez:no-ai-slop`
   (the Skill tool in Claude Code) in detect mode over the prose inside the comments,
   per "Invoked from another skill" in that skill. It reads the wording; you have
   just read the structure. Its findings join the audit as
   `path:line · SLOP · <pattern> · <fix>`. Prefer the repo's own prose skill if
   it has one, and say which you ran.
4. **Print the audit before editing.** One line per finding:
   `path:line · RULE · what's wrong`. Group by file, keep each line short. Close
   it with a `DELETE` section listing every comment you intend to remove and
   why, and a `TRIM` section for every one you intend to compress. An audit with
   neither section is a claim that the file carries no fat — make it only when
   that is true. Do not ask for approval; the audit is the record, not a gate.
5. **Apply the fixes** with Edit.
6. **Verify:** re-run the linter to zero errors if there is one, then the repo's
   formatter and its analyzer or type-checker over the touched files.
7. **Summarize:** counts by rule, lines deleted and lines compressed away, plus
   anything left for the user (a bare `TODO:` with no issue to reference, a stale
   comment you could not confirm, a rule the repo layer overrode). Leave every
   edit unstaged and close by naming `/ez:commit`; a comment pass over
   pre-existing code is its own `chore` commit.

## Doc comments (D)

- **D1** — A doc comment on every language construct, **public and private** —
  classes, enums and their values, mixins, traits, interfaces, constructors,
  fields, properties, methods, top-level functions, type aliases, constants.
- **D2** — Open with the identifier it documents, then a third-person verb.
  `parseHeader returns …`, not `Parses …`. Where the language's doc tool has a
  reference syntax, use it on that opening identifier.
- **D3** — Mark every reference to another identifier in prose with the
  language's reference syntax. A name that does not resolve in scope takes
  backticks instead, so the doc tool does not raise an unresolved-reference
  warning.
- **D4** — Lead with a one-line summary of what the construct is or does.
  Tooling shows that line alone.
- **D5** — State the **caller's contract** — what to pass, what comes back, what
  constrains the call. Cut internal provenance: which helper computes the value,
  which layer it resolves through. It is immaterial to callers and goes stale.
- **D6** — Don't document conventional overrides of well-known members —
  equality, hashing, string conversion, `Dispose`, `__repr__`, a framework's
  lifecycle methods. The base class or protocol documents the contract.
- **D7** — Complete sentences ending with a period, wrapped at the repo's line
  width (80 columns unless the repo says otherwise, indentation included).
  Continuation lines are plain prose; don't repeat the leading identifier.
- **D8** — **A summary plus at most two NOTEs.** The summary is 1-2 sentences and
  three lines at the outside; each NOTE is 1-2 lines; nothing sits between them.
  The bloat is never the summary — it is the middle paragraphs, which are X2
  rationale wearing a doc comment. Route them to the PR description. Take the
  second NOTE only for a second caveat a caller must know, never to keep
  rationale that lost its paragraph.

## Freestanding comments (C)

- **C1** — Line comments inside bodies, and for file or section headers.
- **C2** — A file header sits above the imports. Where the language distinguishes
  a module doc comment from a plain comment, don't leave a module doc comment
  dangling above a file that declares no module.
- **C3** — Local functions and closures take a plain line comment with a
  third-person verb and **no** identifier-reference syntax on the opening word.
  They are usually outside the doc tool's scope, so the reference form would not
  resolve. Mark their parameters in prose as usual.
- **C4** — A body comment sits directly above the lines it explains and runs 1-2
  lines. Longer than that belongs in the PR description.

## NOTE (N)

- **N1** — `NOTE:` marks a constraint a reader must know before touching the
  code — a platform limitation, an invariant the type cannot enforce, a "must be
  called from X", a known inefficiency. It is not a place to park rationale.
- **N2** — In a doc comment it is its **own final paragraph**, set off by a blank
  comment line. Never inline it into the summary.
- **N3** — In a body it sits directly above the code it qualifies, with a blank
  line above unless it opens the block.
- **N4** — The body is a capitalized complete sentence.

```python
def list_active_notifications():
    """list_active_notifications returns every currently displayed
    notification as a record of its ID and content.

    NOTE: This may not be supported on all platforms.
    """
```

## TODO (T)

- **T1** — A bare `TODO:` is a last resort. Flag every one you find in the audit
  as needing a tag.
- **T2** — The issue tracking the work is the only valid tag — `TODO(#83):` — and
  the body is a capitalized sentence. Anything else (a roadmap phase, a
  milestone, a person) orphans the moment the thing it names closes, which is the
  failure this rule exists to prevent. Open the issue before writing the TODO
  rather than leaving it bare.
- **T3** — A TODO that qualifies an existing explanatory comment is appended to
  the end of it. Otherwise it is its own comment directly above the code.
- **T4** — Prefer a body comment. A TODO in a doc comment is rare and needs to be
  something a caller cares about.

## Given / When / Then (G)

Tests are annotated by **logical atom, not by phase**. Each distinct piece of
setup, each action, and each assertion cluster gets its own one-line, full
sentence comment, capitalized after the colon, naming the section it belongs to.
Repeating a phase is the norm, not a smell — a second `// Given:` is the code
equivalent of the `And` line in
[the original formulation](https://martinfowler.com/bliki/GivenWhenThen.html),
which is how Gherkin combines multiple expressions within one clause.

None of this depends on the language or the test framework. The annotation is
`<line comment token> Given: <sentence>`, so it reads `# Given:` in Python or
shell and `// Given:` in a C-like language, and everything below applies to both.

- **G1** — An atom is classified by what it *does* — the state of the world
  before (Given), the behavior under test (When), an expected change after
  (Then) — not by where it happens to sit in the body.
- **G2** — **One atom per annotation.** A single `# Given:` over a block that
  builds a user, then two orders, then a frozen clock is four givens, not one.
  Split it and let each line tell that atom's story. This is the most common
  violation anywhere, and the one to look for first in any test file.
- **G3** — **One line each.** Never wrap an annotation. A line that will not fit
  is almost always covering two atoms, so split it rather than wrapping it.
- **G4** — An annotation names what the atom *means*, not the API it calls.
  `# Given: A task completed twice last week.` — not
  `# Given: create_completion is called twice.`
- **G5** — Omit `When:` when the Given *is* the action, as in a pure render or
  construction test.
- **G6** — Don't invent atoms the body doesn't show. A helper call that hides
  several steps behind one line takes one annotation summarizing its effect.
- **G7** — Rationale that does not fit the beat goes in a separate,
  non-annotation comment above the test, in a `NOTE:` paragraph below the
  annotation (G9), or short and inline on the specific line it explains. Never
  inside the annotation, and never stacked above it.
- **G8** — The colon after `Given`/`When`/`Then` is a sanctioned marker, not a
  violation of P1.
- **G9** — **An annotation opens its comment block.** Stacking rationale above a
  `# Then:` is the standing dodge around G3 — it buys three lines of prose while
  the annotation stays technically short, and it buries the phase story a reader
  scans for. Lead with the annotation; anything more trails it as a `NOTE:`
  paragraph set off by a blank comment line. Most stacked rationale is better
  deleted than relocated — if the annotation and the code already carry it, the
  NOTE is a third telling. Prefer one comment above the test case over a NOTE
  per assertion.

✗ Stacked above, and covering two atoms:

```python
# The gate used to read the instant value, which a migrated field never joins,
# so it would compare None against "" and always pass.
# Given: A migrated field and a submitted form.
form = migrated_form(); form.submit()
```

✓ Annotation first, one atom each, rationale trailing as its own NOTE:

```python
# Given: A form whose only field was migrated from the legacy schema.
form = migrated_form()

# When: The form is submitted.
form.submit()

# Then: Submitting is refused.
#
# NOTE: The gate used to read the instant value, which a migrated field never
# joins, so it would compare None against "" and always pass.
assert form.submit_enabled is False
```

## Punctuation and prose (P)

- **P1** — **No `:` as sentence structure.** The `label: explanation` form is the
  most common violation anywhere. The sanctioned exceptions are a real list and
  the `Given:`/`When:`/`Then:`/`NOTE:`/`TODO(#83):` markers. Restructure into a
  full sentence, split into two, or join with `;`.
- **P2** — Dashes are `ez:no-ai-slop`'s call, under the budget in that skill;
  none in a comment that fits on two lines.
- **P3** — Third person, present tense.
- **P4** — Wrap at the repo's line width, indentation included.

## Section banners (B)

**Only if the repo layer declares them.** Most repos have no banner convention,
and inventing one is a large, unrequested diff. When the repo does declare one,
reproduce it exactly — width, label vocabulary, padding, and which declarations
earn which form — and generate each banner rather than hand-counting characters.
A banner never carries prose; anything worth saying goes in the doc comment
below it.

## Deleting comments (X)

Delete, don't preserve. The bar is that a reader would miss the comment if it
were gone; anything under that bar is bloat, however true it is. Report each
deletion in the audit with its reason. Nothing here is language-specific.

- **X1** — Restates the adjacent code and adds nothing. A doc comment that only
  re-reads the signature back — parameter names, the return type, the obvious
  verb in the identifier — is the commonest form.
- **X2** — Design rationale, history, sequencing, or "why we deferred this"
  narrative. That belongs in the PR description, not in source. Planning
  vocabulary is the tell — a roadmap phase, a slice, a milestone, "the approach
  we settled on", a comparison to the thing it replaced.
- **X3** — Internal provenance inside a doc comment (see D5). Trim the clause;
  keep the contract.
- **X4** — Obsolete. It describes code, a workaround, or a constraint that no
  longer exists. Confirm against the current file before removing; an obsolete
  comment is worse than none, since a reader trusts it.
- **X5** — Duplicates a fact already stated by a nearby comment, by the
  declaration's own doc comment, or — the case most often missed — by a comment
  in another file. An implementation's doc, its test's `NOTE:`, and that test
  file's header all telling the same story is one telling and two echoes. Keep
  the one closest to the code that owns the fact and cut the rest, or point at it
  if the reader really does need it from both places.
- **X6** — Costs more lines than it returns. The point is real, but it takes six
  lines to make and two to say. `ez:no-ai-slop` decides how far to compress;
  this entry exists so the audit can cite it. Compress rather than cut, and
  count the compression as a `TRIM`.
- **X7** — Explains the language or the framework rather than this code. The
  reader is fluent in both; narrating what a well-known construct does is bloat.
  What earns its place is the surprise — what the framework *doesn't* do here, or
  the reason the obvious call is the wrong one.
- **X8** — **Describes transient state the committed tree never held.** A
  discarded attempt, what the working tree looked like mid-change, or what the
  code did before this commit. A reader arrives at the file as it is and has no
  referent for any of it, so the comment reads as being about code they cannot
  find. The tell is a past-tense verb whose subject is not on screen — "hand-
  rolling X produced", "this used to be", "the first attempt", "before the
  rewrite", "we tried". Distinct from X4, which a change *left behind*; this is
  one the change *created*, describing the edit rather than the result. Keep
  whatever invariant the story was evidence for and drop the story; a comment
  reflects the code below it.

Keep anything that records a gotcha, an "X not Y because…", an issue or spec
reference, or an invariant the code cannot express. When genuinely unsure whether
a comment is load-bearing, tighten it instead of cutting it and say so in the
summary. Uncertainty is a reason to compress, not a reason to leave a comment
untouched.

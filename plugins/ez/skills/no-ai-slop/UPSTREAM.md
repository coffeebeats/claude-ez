# Upstream

`SKILL.md` and `eval.md` are vendored from
[petergyang/no-ai-slop](https://github.com/petergyang/no-ai-slop) and edited to
sit under the `ez` chain. Vendored rather than installed with
`npx skills add --global`, so the local edits below survive an upstream release.

- Pinned commit: `000650b` (2026-09-02), path `skills/no-ai-slop`.
- Upstream ships no `.claude-plugin`, only a Codex manifest, so there is no
  marketplace to depend on and no `dependencies` entry that would replace this
  copy. Its `skills/<name>/` layout is already the Claude plugin layout, so the
  directory drops in unchanged.
- Invoked as `/ez:no-ai-slop`. The plugin namespace keeps it from colliding with
  a repository's own `no-ai-slop`, so both can be loaded at once.

## Local edits

- Added "How this skill fits the chain", which gives whatever the calling
  context enforces mechanically precedence over prose judgment, and states the
  three carve-outs.
- Added the "A project skill wins" deferral, so a repo with its own prose skill
  gets that one instead.
- Rewrote the voice-preservation rules to name the house voice instead of an
  author persona, keeping upstream's instruction for genuinely authored prose.
- Replaced the em dash rule with a budget of one per paragraph in a PR body and
  none in a short comment.
- Added "Invoked from another skill" for `/ez:comments`, `/ez:pr`, and
  `/ez:publish`.
- Dropped the audience-and-venue question, which the calling skill already fixes.
- Named AGENTS.md beside CLAUDE.md in the description, so Codex sessions
  route their instruction files here too.
- Dropped upstream's colon-case and mandated-repetition eval checks, and added a
  carve-outs section.

Upstream's `b53e265..000650b` made two changes. Its workflow step 2 dropped the
"3-5 voice signals" count and the keep-it-internal instruction; this fork had
already cut that step further, and upstream's replacement trait list is the
personal-voice enumeration the house-voice carve-out exists to override, so
nothing was taken. Its `eval.md` dropped the process self-check from **Final
read**, which applied here and was taken.

A repository may carry its own fork of the same upstream at
`.claude/skills/no-ai-slop/`. Where one does, re-sync both or neither; a fork
tuned to a repo differs from this copy mainly in naming the rules its own linter
enforces, which this copy leaves to the caller.

## Re-syncing

The edits above are extensive, so overwriting from upstream would silently drop
them. This directory is not itself inside a git repository, so `git apply -3`
has nothing to three-way against. Read upstream's own diff across the two
commits and port each hunk by hand instead, which is the right granularity
anyway — the diffs are small and each one has to be judged against the carve-outs.

```sh
for f in SKILL.md eval.md; do
  for ref in 000650b HEAD; do
    gh api "repos/petergyang/no-ai-slop/contents/skills/no-ai-slop/$f?ref=$ref"       --jq .content | base64 -d > "$ref-$f"
  done
  diff -u "000650b-$f" "HEAD-$f"
done
```

For each hunk, decide whether it touches something a local edit deliberately
overrode; take it only where it does not. Then re-read the "Local edits" list to
confirm each still holds, and update the pinned commit above.

## Licence

MIT, Copyright (c) 2026 Peter Yang. The full text is in `LICENSE`, alongside
this file; it must travel with any copy of `SKILL.md` or `eval.md`.

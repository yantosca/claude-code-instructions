# Validating documentation against code and config files

Method for checking whether a project's documentation still matches
its actual source code and configuration — i.e. finding undocumented
new features, and doc content that's stale, incomplete, or outright
wrong relative to what the code currently does. This is a companion to
[ReadTheDocs-Sphinx-documentation-validation.md](ReadTheDocs-Sphinx-documentation-validation.md)
(which covers build hygiene) — that file's step 3 is a one-line pointer
to this fuller method. Works on any repo with both a docs tree and real
source/config, independent of language or doc toolchain.

## 1. Start from the changelog, not a blank search

Don't try to guess what might be undocumented from scratch. Read the
project's `CHANGELOG.md` (or release notes) for the last several
releases and pull out every `Added`/`Changed`/`Removed` entry that
names a concrete, checkable thing: a new config key, function,
extension/plugin, CLI flag, output/diagnostic name, or a rename. This
turns an open-ended search into a bounded checklist you can verify item
by item.

## 2. Read the actual code for each item — don't trust the changelog prose alone

A changelog summary can itself be imprecise, or describe the change at
too high a level to tell whether the docs are still accurate. For each
candidate item, grep the source for the exact identifier the changelog
names and read enough of the surrounding code (or the diff that
introduced it, via `git log -S"<identifier>"` / `git show`) to know
precisely what it does now — required vs. optional, what triggers an
error, what the current mechanism actually is. Only then check the
docs against that ground truth, not against the changelog's wording.

## 3. Classify each item, don't just mark it "missing"

For every candidate item, grep the docs tree for it and sort into:

- **Missing** — no mention anywhere.
- **Stale/wrong** — mentioned, but the text describes an older
  mechanism, a removed feature, or a superseded name. This is worse
  than missing (it actively misleads) and just as important to catch.
- **Adequate** — mentioned and still accurate.
- **Not doc-worthy** — a real code change that's a pure internal
  implementation detail (e.g. an argument that only fixes how an
  existing operation is computed internally) sitting below the
  abstraction level the docs already operate at. Don't manufacture a
  gap here just because something in the changelog has no doc mention
  — check whether anything comparable is documented at that level of
  detail elsewhere first.

## 4. For renames/removals, grep the whole tree for the old name — in both directions

When the changelog says "renamed X to Y" or "removed Z": grep the
*entire* docs tree for `X`/`Z` (not just the pages you'd expect) to
catch stale references anywhere, and also grep the *current source*
for the same old name, to confirm the code itself is fully consistent
and you're not "fixing" docs to match code that's still inconsistent
elsewhere.

## 5. Base every concrete detail in a fix on a real example already in the repo

When writing corrected doc text (an example config value, a function
name, a numeric ID, an option string), don't invent a plausible-looking
value. Grep for how an existing, still-correct doc entry or a real
sample config/fixture already shows the same kind of thing, and mirror
that exact pattern/value. If no real example exists to copy, say so
rather than filling in a guess.

## 6. Never fabricate a fix for something you can't verify — flag it instead

Some gaps have no derivable correct answer from the repo: a missing or
placeholder citation with no matching bibliography entry and no header
comment naming the real source, for example. Don't invent a
plausible-sounding citation, URL, or value. List these separately as
"needs human input" and leave the text untouched.

## 7. Fan out with a concrete, numbered checklist — not an open-ended prompt

If there are more than a handful of candidate items, delegate the
cross-check to parallel subagents, but hand each one the exact
checklist from step 1 (item text, which doc file(s) to grep, which
source file/lines to read) rather than "find undocumented features."
This keeps each finding independently reproducible and makes the
report structured per-item instead of free-form prose.

## 8. Verify every finding yourself before fixing or reporting it

Subagents (and a first read-through in general) can shift line
numbers or misquote. Before acting on any finding — yours or a
subagent's — re-grep/re-read the exact file and line to confirm it
still says what's claimed, and re-check the source-code half of the
claim the same way.

## 9. When the doc source is shared across multiple sibling projects

If the docs tree includes a shared/vendored subtree (a common
submodule, a shared template) used by more than one sibling repo, and
only some of its content applies to the repo you're auditing, prefer
the least invasive, most certain fix for any cross-references that
point at intentionally-excluded content — e.g. drop a broken
cross-reference to plain text — over guessing at a specific external
URL. Only add a real hyperlink if you find and confirm an existing
precedent link elsewhere in the same docs tree; mirror it exactly
rather than constructing a new URL from a guessed slug.

## 10. Re-verify after fixing

After applying fixes, grep the docs tree once more for every stale
term/old name you just fixed, to confirm no other occurrences were
missed, and rebuild the docs (see the companion Sphinx file) to
confirm no regressions were introduced.

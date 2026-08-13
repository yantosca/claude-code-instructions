# Validating Sphinx/ReadTheDocs documentation

Method for auditing a Sphinx/reST documentation repo (or the `docs/`
tree of a code repo) for build errors, stale content, and hygiene
issues. This documents the *method*, not a snapshot of one repo's
findings — re-run the checks fresh each time.

## 1. Build the docs for real — don't just grep for syntax

Sphinx/docutils warnings (title underlines too short, malformed
directive options, undefined `:ref:` labels, bad list-table options,
etc.) are easy to miss by eye but trivial to catch by actually
building:

```bash
cd docs   # or wherever conf.py's parent is
sphinx-build -b html -a -E source build/html --keep-going
```

- `-a -E` forces a full rebuild ignoring the cached environment —
  without it, Sphinx skips unchanged files on a second build and
  **hides warnings** from anything not touched since the last build.
  Always use `-a -E` when auditing, even on a repo that was "just
  built."
- `--keep-going` surfaces every error in one pass instead of stopping
  at the first one.
- If a dependency (`myst-parser`, `recommonmark`, `sphinxcontrib-bibtex`,
  etc.) is missing, install from the repo's pinned `docs/requirements.txt`
  rather than latest — version drift between the pin and what's
  installed locally can itself hide/introduce spurious warnings (e.g.
  a `html_theme_options` key that's valid in the pinned theme version
  but rejected by a newer one installed locally). If pip refuses due to
  an externally-managed environment, `pip install --user --break-system-packages
  <pkg>` is fine for a throwaway doc-build tool — it's not modifying
  the project.
- Fix each warning at its root cause: extend title underlines/overlines
  to match title length, fix `:ref:` targets to the label that actually
  exists (`grep -rn "^\.\. _label:"`), fix directive options against the
  real docutils/Sphinx spec (e.g. `list-table` takes its caption as a
  positional argument, not a `:caption:` option — there's no such
  option).

## 2. Check for content typos, grammar, and factual inconsistencies

Sphinx won't catch prose problems. Delegate a close-reading pass to a
subagent (cheaper than doing it inline, and it won't inherit your
priors about what the text "should" say): give it every `.rst`/`.md`
page, ask it to report only high-confidence typos, grammar errors,
and — importantly — **internal inconsistencies spotted by reading
closely**, e.g. a page asserting "X is not supported" while another
page in the same site has a full section documenting X. Always verify
each finding against the actual file before fixing (subagents
occasionally shift line numbers or misquote).

## 3. Cross-check docs against the actual codebase for undocumented features

When the repo has real source code alongside the docs (not a docs-only
repo), don't assume the docs are complete just because they build
clean. Check recently changed/added code (`git log --stat` on the
relevant window, or a diff against the last doc-review point) for new
CLI flags, config keys, functions, directives, or file formats, then
grep the docs tree for whether each one is mentioned anywhere. Flag
anything real in the code with no doc coverage — new features silently
missing from user guides are a common gap that a docs-only build check
can't surface.

## 4. Toctree hygiene: remove/fix stale entries

A `.rst` file's own build warnings say nothing about whether it's
still correctly wired into navigation. Separately:

- Diff every path listed under a `.. toctree::` directive (typically in
  `index.rst`, but nested toctrees exist too) against files that
  actually exist on disk — a toctree entry pointing at a renamed or
  deleted `.rst` file is a hard Sphinx error, not a warning.
- Conversely, check for any `.rst` file under `source/` that isn't
  reachable from any toctree — Sphinx normally warns
  `document isn't included in any toctree` for these, which the full
  rebuild in step 1 will surface; remove the toctree entry (or the
  orphaned file, depending on which is stale) rather than leaving it.

## 5. Anonymize inline hyperlinks (`` `_ `` → `` `__ ``)

Convert named inline hyperlinks to anonymous ones to preempt "Duplicate
explicit target name" warnings:

```python
import re, glob
pat = re.compile(r'(`[^`]*<[^>]+>`_)(?!_)')
for f in glob.glob('**/*.rst', recursive=True):
    text = open(f).read()
    new_text, n = pat.subn(r'\1_', text)
    if n:
        open(f, 'w').write(new_text)
```

- **Only** convert the inline explicit form `` `text <url>`_ `` (URL
  embedded directly in the same construct). Do **not** touch bare
  references like `` `label`_ `` that resolve to a separate
  `.. _label: url` target defined elsewhere — those are meant to be
  reused by the same name and converting them to `__` breaks the
  reference (anonymous refs can't point at a named target). Check for
  this case first: `grep -rn "^\.\. _[A-Za-z0-9_-]*: http"` across the
  docs tree; if it's empty, every `` `_ `` in the tree is the inline
  form and the blanket regex above is safe.
- Why this matters even when no warning currently fires: docutils only
  warns on duplicate target names when the same link *text* resolves to
  **different** URLs. Two identical `` `Text <same-url>`_ `` links in
  one document currently build silently — but the moment someone edits
  one of the URLs and not the other, it becomes a real warning.
  Anonymizing preemptively avoids relying on both copies always staying
  in sync. (Verified this docutils behavior directly with a minimal
  `sphinx-build` repro rather than assuming it — worth doing again if
  the behavior ever seems to contradict what's expected, since docutils
  version drift could change it.)

## General notes

- Prefer fixing the root cause over suppressing the warning (e.g. don't
  add `-Q`/`nitpick_ignore` to silence an undefined `:ref:`— fix the
  label).
- After every fix, re-run the full `-a -E --keep-going` build again to
  confirm zero warnings before considering the pass done.

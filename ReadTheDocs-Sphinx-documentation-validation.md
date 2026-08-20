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
- If the docs use Sphinx `autodoc`/`autosummary` against the repo's
  *own* Python package (an `api.rst`-style page recursively
  autosummarizing `mypackage.*`), a generic docs-only environment will
  fail with `ExtensionError: ... no module named mypackage` even with
  every Sphinx package correctly pinned — `autodoc` has to actually
  `import` the package to introspect it. Check whether the project's
  own dev/conda environment (not just a shared docs-build env reused
  across sibling repos) already bundles the matching Sphinx pins *and*
  has the package installed/importable (`python -c "import mypackage"`)
  — that combined environment is what building this kind of repo
  needs, whereas a plain docs-only env only works for repos whose docs
  are static `.rst`/`.md` content with no autodoc against local code.
- Fix each warning at its root cause: extend title underlines/overlines
  to match title length, fix `:ref:` targets to the label that actually
  exists (`grep -rn "^\.\. _label:"`), fix directive options against the
  real docutils/Sphinx spec (e.g. `list-table` takes its caption as a
  positional argument, not a `:caption:` option — there's no such
  option).
- The "N warnings" summary count can include docutils messages at
  **ERROR** severity too (a malformed grid table, an unparseable
  directive block), which won't match a naive `grep "WARNING"` over
  captured output — if the summary count and your grep count disagree,
  don't assume the grep is complete. Re-run with `-w <file>` (e.g.
  `sphinx-build ... -w /tmp/sphinx_warnings.txt`) to get every
  diagnostic in one place regardless of its severity label.
- By default Sphinx does **not** warn on an unresolved `:func:`/`:mod:`/
  `:class:` (or other `py:*`) cross-reference — it just renders as
  plain unlinked text, silently. To actually catch broken object
  references (see step 7), add `-n` (nitpicky mode) to the build
  command; this surfaces a large amount of unrelated noise too (short
  type-hint aliases like `xr.Dataset` in numpydoc-style docstrings that
  napoleon turns into unresolvable `:class:` roles) unless the project
  also configures `intersphinx_mapping` and `napoleon_type_aliases` —
  filter the output to just the targets you're actually checking rather
  than trying to zero out every nitpicky warning.

## 2. Check for content typos, grammar, and factual inconsistencies

Sphinx won't catch prose problems. Delegate a close-reading pass to a
subagent (cheaper than doing it inline, and it won't inherit your
priors about what the text "should" say): give it every `.rst`/`.md`
page, ask it to report only high-confidence typos, grammar errors,
and — importantly — **internal inconsistencies spotted by reading
closely**, e.g. a page asserting "X is not supported" while another
page in the same site has a full section documenting X. Always verify
each finding against the actual file before fixing — see
[Validating-documentation-against-code.md](Validating-documentation-against-code.md)
step 8 for why (subagents occasionally shift line numbers or misquote).

## 3. Cross-check docs against the actual codebase for undocumented features

When the repo has real source code alongside the docs (not a docs-only
repo), don't assume the docs are complete just because they build
clean. Check recently changed/added code (`git log --stat` on the
relevant window, or a diff against the last doc-review point) for new
CLI flags, config keys, functions, directives, or file formats, then
grep the docs tree for whether each one is mentioned anywhere. Flag
anything real in the code with no doc coverage — new features silently
missing from user guides are a common gap that a docs-only build check
can't surface. For the fuller method (starting from the changelog,
telling "missing" apart from "stale/wrong", handling renames, and
verifying fixes) see
[Validating-documentation-against-code.md](Validating-documentation-against-code.md).

Pay special attention to **verbatim reproduced console output or
interactive-menu text** — a sample `cmake` configure transcript, a
script's numbered `Choose simulation type:`-style prompt copied into a
doc page. Grepping for keyword presence won't catch these going stale,
since the keywords (option names, menu item text) can all still exist
elsewhere in the doc while the *specific reproduced block* silently
drifts: a new choice added to the real script but missing from the
transcript, choices reordered/renumbered, or (seen in practice) a
phantom choice in the doc that was removed from the script long ago.
Diff each such block line-by-line against the actual script's
print/`printf` statements (or a fresh real run of the tool) rather than
trusting that the surrounding prose is still accurate.

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

## 6. Option-group hygiene: default first, right directive for booleans

Pages that document a set of mutually-exclusive settings (CMake build
switches, YAML config keys, CLI flags) commonly use Sphinx's
`.. option::` directive once per accepted value, with prose noting
which one is `"(Default option)"` / `"This is the default setting"`.
Two recurring problems in this pattern:

- **The default isn't listed first.** Readers scan top-to-bottom
  expecting the first-listed value to be the default; when the
  default is buried second (or later) among several choices, that
  expectation is silently violated. Grep the whole file for `default
  option`/`default setting`/`recommended option`/`recommended
  setting` and, for each hit, check whether it's already the first
  `.. option::`/`.. describe::` sibling under its heading — reorder if
  not. Watch for settings whose default is *conditional* (e.g. `true`
  by default for one simulation type, `false` for another, as shown in
  a code example elsewhere on the page) — there, whichever value
  matches the code example's default stays first; don't reorder those.
- **Plain booleans should use `.. describe::`, not `.. option::`.**
  `.. option::` is meant for named/CLI-style values (e.g. `fullchem`,
  `Release`, a resolution string); a bare `y`/`n` or `true`/`false`
  toggle is more accurately a `.. describe::` (Sphinx's generic
  definition-list directive — no special indexing/cross-reference
  semantics). Before converting, grep for `` :option:`y` `` /
  `` :option:`n` `` / `` :option:`true` `` / `` :option:`false` `` (or
  whatever the boolean tokens are) across the whole docs tree — if
  nothing cross-references them by that role, the rename is safe.

For a file with many instances of either problem, don't fix each one
with an individual find-and-replace: first do a blanket, anchored
`sed` pass converting every plain-boolean `.. option:: true`/`.. option::
false` (or `y`/`n`) to `.. describe::` in one shot (safe once the
cross-reference check above is clean, and doesn't touch line counts,
so any line numbers you gathered beforehand stay valid) — then apply
targeted reorder edits only to the subset of blocks where the default
wasn't already first. This is much cheaper than hand-editing every
occurrence individually, and reordering already-correct blocks would
be a no-op edit anyway. While reading through each block to check
order, you'll sometimes find an adjacent, unrelated bug (e.g. two
sibling values both accidentally labeled `true`, or a description with
the wrong verb) — fix the ones that block the requested change from
being meaningful (e.g. a duplicate label makes "which one is the
default" ambiguous), but otherwise flag-don't-fix content bugs that
are outside what was asked, the same as any other unrelated finding.

## 7. `:func:`/`:mod:`/`:class:` role consistency (Python autodoc projects)

When a project cross-references its own API with Sphinx's Python
domain roles, it's common for `:func:` to get used as a catch-all for
every dotted name — including ones that are actually modules or
classes, not functions. This is easy to do (the roles all *render* the
same way, as a plain code-styled link) and easy to miss by eye, but
Sphinx's Python domain distinguishes them internally, and picking the
wrong one is a real inconsistency worth fixing.

- **Find the candidates**: grep the docs tree for every
  `` :func:`...` ``, `` :mod:`...` ``, `` :class:`...` `` target and
  extract the unique dotted names. A large skew (e.g. 100+ `:func:` to
  a handful of `:mod:`) is itself a signal that `:func:` is being used
  as a default rather than deliberately.
- **Classify each one against the real package**, don't guess from the
  name: import progressively shorter prefixes of the dotted path until
  one succeeds, then `getattr` the remaining parts, and check the
  result with `inspect.isfunction`/`inspect.isclass`/`isinstance(obj,
  types.ModuleType)`. A bare `mypkg.subpkg.some_script` with no
  trailing function name is almost always a module (especially for a
  standalone script meant to be run via `python -m`), while
  `mypkg.subpkg.some_module.some_function` is a function.
- **This same introspection pass surfaces genuinely broken paths for
  free** — do this check even if you weren't asked to, since fixing
  the role is pointless if the path doesn't resolve anyway. Real
  examples found this way in one audit: a missing letter in a package
  segment (`module` vs `modules`), a `/` typo'd in place of a `.`, a
  reference to the wrong sibling module name, a wrong top-level package
  name entirely (copy-paste from a sibling project), and a missing dot
  merging two path segments into one. All of these fail silently in
  non-nitpicky mode (see step 1) — only `-n` surfaces them, and even
  then as "reference target not found" rather than anything
  pointing at the typo directly.
- **Apply the mechanical, always-`:mod:` fixes in bulk**: once you have
  the confirmed list of dotted names that are modules, do one
  script/pass per exact name (`` :func:`name` `` → `` :mod:`name` ``)
  across every `.rst` file rather than editing occurrence by occurrence
  — same rationale as the option-group bulk-sed approach in step 6.
  Handle the broken-path cases and any `:class:`-not-`:func:` cases
  (e.g. a third-party class like `xarray.DataArray` tagged `:func:`)
  as individual targeted fixes, since each needs a different
  replacement string.
- **Watch for collateral damage in fixed-width content**: if a
  correction changes the *length* of visible text inside a
  `.. table::`/grid-table block (e.g. fixing `modules` from a
  shorter typo adds a character), the row's `|` column no longer lines
  up with the table's border row, which docutils reports as `ERROR:
  Malformed table` — re-run the build (see step 1's note on `-w`) after
  any edit inside a grid table, not just after prose edits.

## General notes

- Prefer fixing the root cause over suppressing the warning (e.g. don't
  add `-Q`/`nitpick_ignore` to silence an undefined `:ref:`— fix the
  label).
- After every fix, re-run the full `-a -E --keep-going` build again to
  confirm zero warnings before considering the pass done.

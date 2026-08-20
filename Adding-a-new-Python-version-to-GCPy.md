# Adding support for a new Python version to GCPy

Method for adding a new Python version (e.g. 3.15, once 3.14 is
established) to GCPy: verifying it's actually possible, creating the
environment file, updating docs/CI, catching runtime bugs a solver
can't see, and updating the downstream `geoschem-gcpy` conda-forge
feedstock. Distilled from adding Python 3.14 support.

## 1. Verify feasibility with a real dry-run solve, not a doc claim

Don't trust an existing "we can't build a Python N environment yet"
note in the docs at face value — the conda-forge packages it blamed
(e.g. `esmf`/`esmpy`/`xesmf`) may have been rebuilt since. Check for
real:

```bash
mamba env create --dry-run -n test --file=<unpinned-list-of-the-same-packages>
```

with the target Python version and every other package **unpinned**
(just names, no `=version`) so the solver picks whatever's current on
conda-forge. If it solves, the blocker is gone; if not, the error
lists exactly which package still conflicts and why — that's your
real, current constraint, not last quarter's.

## 2. Watch for the free-threaded build trap

If the new Python version has a free-threaded variant (CPython's
`cp3XXt` build tag), an unpinned `python =3.XX` spec may resolve to it
by default. Check what the solver actually picked
(`grep "python " <dry-run output>` for the build string) — if it says
`_cp3XXt`, that's free-threaded. Most of the scientific Python stack
(`cftime` and other C extensions) hasn't declared free-threading
support yet, and running under that build silently re-enables the GIL
with a `RuntimeWarning` at every affected import. Pin the standard
build explicitly instead: `python =3.XX=*cp3XX` (note the conda
matchspec glob `*cp3XX`, not `*cp3XXt`) — verify with another dry-run
that this resolves to the non-t build string.

## 3. Create the environment file from the dry-run's exact resolved versions

Copy the existing `docs/environment_files/gcpy_environment_py3XX.yml`
(same structure, same comments, same package list and order) and fill
in versions from the dry-run output — not "whatever's latest when I
write this," since that drifts the moment you commit it. Then run the
dry-run *again* against the final pinned file itself (not just the
unpinned test) to confirm the exact pins you wrote actually solve
together.

## 4. Actually build it and run the examples — the solver can't see runtime bugs

A successful dependency solve only proves the packages are
*installable* together, not that the package's own code works under
the new interpreter. Build the environment for real and exercise a
few example scripts (particularly ones with `python -m
package.examples.*` entry points, and anything importing C-extension
heavy libraries) before believing the new version is supported. Two real,
unrelated GCPy bugs surfaced this way that the dry-run gave no hint
of:

- The free-threaded-build GIL warning from step 2 (only visible at
  runtime import, not at solve time).
- A pre-existing packaging anti-pattern: `gcpy/examples/*/__init__.py`
  files did `from .some_script import *`, so importing the *parent*
  package (which `python -m pkg.sub.script` must do first) silently
  pre-loaded the target script as a side effect. `runpy` warns about
  this ("found in sys.modules... this may result in unpredictable
  behaviour") on *any* Python version — it just took actually running
  a script via `-m` on the new environment to notice. Worse, one
  script (`create_test_plot.py`) had a latent bug this was masking: it
  did bare `import gcpy` then called `gcpy.plot.single_panel(...)`,
  which only worked because a *sibling* script's wildcard-imported
  `from gcpy.plot.single_panel import single_panel` had already
  registered `gcpy.plot` as a module attribute as a side effect.
  Removing the wildcard imports (the correct fix for the runpy
  warning) exposed the AttributeError. Fix both: empty the `__init__.py`
  files (a working example already existed in the repo — the `gcst`
  subfolder's `__init__.py` was already just a docstring), and fix any
  script whose imports turn out to have been relying on the
  now-removed side effect.

## 5. Update docs and CI to match

- Add a "Version (Python 3.XX)" column to the dependency table in
  `docs/source/Getting-Started-with-GCPy.rst`, and rewrite any prose
  that claimed the new version wasn't buildable.
- Add `.github/workflows/build-gcpy-environment-py3XX.yml`. If sibling
  workflow files for other Python versions have diverged from each
  other (e.g. one has an extra smoke-test step the others lack), copy
  the more complete one rather than whichever is easiest to diff
  against — don't propagate a gap.
- Add `CHANGELOG.md` entries for the new environment file, the new CI
  workflow, and any bugs fixed along the way.

## 6. Update the downstream conda-forge feedstock too

If the package has a conda-forge feedstock (a separate repo, e.g.
`geoschem-gcpy-feedstock`), the new Python version isn't actually
installable via `conda install` until the feedstock is updated:

- Check `recipe/meta.yaml`'s `build: skip:` selector for an explicit
  `py>=3XX` (or similar) exclusion — remove it once feasibility is
  confirmed (step 1). Also bump any hard-pinned sub-dependency
  versions in `host`/`run`/`test` (e.g. `esmf`/`esmpy`/`xesmf`) to the
  versions confirmed to solve for the new Python version — the
  recipe's existing pins may predate the fix.
- Regenerate the CI build matrix with `conda-smithy rerender`, not by
  hand-editing `.ci_support/*.yaml` — those files encode conda-forge's
  current global pinning (numpy version, docker image, etc.), which
  you can't reliably reconstruct by hand. If the feedstock-maintenance
  tool environment has an old, pinned Python that's too old for the
  current `conda-smithy` release, upgrade both together explicitly
  (`mamba install -n <env> python=3.12 conda-smithy=<latest> -y`) —
  conda/mamba won't move an already-installed Python version on its
  own unless you name the new version directly in the install command.
- **Watch for a new-version bump silently breaking an old one still in
  the matrix.** Bumping a pinned dependency (e.g. `xesmf`) to satisfy
  the new Python version can raise *that dependency's own* minimum
  supported Python — if an older Python version is still in the CI
  matrix, its build will now fail for real (not flakily): the error
  will show the newly-bumped package requiring a higher Python floor
  than the old version being built. The fix is to add a `py<3YY` skip
  condition for the now-incompatible old version (removing it from the
  matrix via `rerender`, which regenerates/deletes the corresponding
  `.ci_support/*.yaml` files), not to fight the solver.
- When rebasing a feedstock branch after `main` has moved (e.g.
  another PR's own rerender was merged), don't try to replay an old
  auto-generated "MNT: Re-rendered..." commit through the resulting
  conflict in generated files — drop it and rerender fresh instead:
  `git rebase --onto main <commit-before-old-rerender> <branch>`, then
  run `conda-smithy rerender` again on the new base. This avoids
  conflict noise entirely, since rerendering regenerates the same
  files from scratch either way.

## 7. Re-verify everything before calling it done

Run the full test suite and a doc rebuild against the *existing*
Python version's environment too (not just the new one) to confirm
none of the fixes (e.g. removing `__init__.py` wildcard imports)
regressed anything for currently-supported versions. Confirm the
feedstock PR's CI is green for every remaining matrix entry, not just
the new version.

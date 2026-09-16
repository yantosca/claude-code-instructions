---
name: gcpy-feedstock-pr
description: Open and land a pull request on the geoschem-gcpy conda-forge feedstock — version bumps, sha256, build numbers, dependency pins, build skips, conda-smithy rerender, and post-merge verification on the channel. Use when asked to update, bump, release, or rebuild geoschem-gcpy on conda-forge, to edit recipe/meta.yaml, to rerender a feedstock, to work out whether a build number needs bumping, or when a feedstock PR's CI or the conda-forge-admin bot is behaving unexpectedly.
---

# Landing a PR on the geoschem-gcpy conda-forge feedstock

Method for changing `conda-forge/geoschem-gcpy-feedstock` — bumping the
gcpy version, adding an interpreter, or repinning a dependency — and
verifying the result on the channel. For the upstream GCPy-side work that
usually precedes a feedstock change, see the `gcpy-add-python-version`
skill; this skill picks up at the feedstock.

## 1. Establish ground truth before editing anything

Three separate things can disagree, and each has to be checked directly.

**Live upstream refs.** `.git/FETCH_HEAD` goes stale, so compare against
the remote itself rather than a cached ref:

```bash
git ls-remote upstream main          # what conda-forge/main really is
git rev-list --left-right --count main...upstream/main
```

**What the channel actually serves.** This is the step most easily
skipped and the one that most often changes the plan. The recipe on
`main` is not evidence of what is published:

```bash
conda search -c conda-forge --override-channels 'geoschem-gcpy=<ver>' --info
```

Read the per-variant pins, not just the version list. A build-string hash
that is *identical* across two package versions means those variants were
built against the same pinned dependencies; a variant with a *different*
hash was built against different ones. That is how you spot a matrix that
was only partially rebuilt.

**conda-forge's current global pinning**, which decides the matrix
regardless of what the recipe says:

```bash
curl -s https://raw.githubusercontent.com/conda-forge/conda-forge-pinning-feedstock/main/recipe/conda_build_config.yaml | grep -A12 '^python:'
curl -s https://api.github.com/repos/conda-forge/conda-forge-pinning-feedstock/contents/recipe/migrations | grep '"name"'
```

The `python:` list and `python_min:` tell you which interpreters exist at
all. The `migrations/` listing tells you whether a version migration is
still in flight or has gone global — which decides whether a local
`.ci_support/migrations/pythonNNN.yaml` in the feedstock is still live or
is now stale. Once a migration is global, a rerender deletes that file on
its own; do not delete it by hand.

## 2. Never push a topic branch to the feedstock repo itself

A `push` event to *any* branch of the feedstock uploads packages to
anaconda.org. The build workflow passes `BINSTAR_TOKEN:
${{ secrets.BINSTAR_TOKEN }}`, and GitHub supplies repository secrets to
`push` events while withholding them from `pull_request` events on forks.
So a branch pushed straight to the feedstock builds *and publishes*, with
no review and no merge.

This has already happened once on this feedstock: a `feature/py314-support`
branch pushed directly published py314 artifacts with pins that existed
nowhere in `main`'s history, and the branch was then deleted. The channel
and `main` stayed out of sync for weeks, and the next bot version bump
would have silently reverted the published pins.

Push to your fork; open the PR from there. If you find this has already
happened, the fix is a PR that brings `main` up to the published state —
and say so explicitly in the PR body, because a reviewer comparing
published artifacts against `main` otherwise has no way to explain the
mismatch.

## 3. Branch off `upstream/main`, not your fork's `main`

```bash
git fetch upstream
git switch -c <topic-branch> upstream/main
```

Your fork's `main` accumulates merged-but-never-upstreamed work and stale
renders (an old `conda-smithy` version, an old pinning timestamp).
Branching off it drags that into the PR. Branching off `upstream/main`
keeps the PR to one reviewable recipe diff.

After a PR lands, resync the fork so the trap is gone:

```bash
git switch main && git reset --hard upstream/main && git push --force-with-lease origin main
```

(conda-forge's docs ask for a fork plus a topic branch. The rerender bot
*can* operate on a PR whose head is the fork's default branch — it has done
so on this feedstock — so branch hygiene is the reason here, not bot
capability.)

## 4. Get the build number right

- Version changed → reset `build: number:` to `0`.
- Version unchanged → **bump** it, or the new artifacts collide with what
  is already published.

Check step 1's `conda search` output before deciding, because a partially
published matrix changes the answer. If a rebuild is needed but a newer
upstream release also exists, bumping the version instead is usually
better than bumping the build number: it makes `number: 0` correct on its
own, and the build-number rebuild would be superseded almost immediately
anyway.

## 5. Edit only `recipe/meta.yaml`

Verify the new hash yourself rather than trusting the PyPI JSON field,
and confirm the recipe's URL template still resolves for the new version:

```bash
curl -sL -o /tmp/sdist.tar.gz https://pypi.org/packages/source/g/geoschem-gcpy/geoschem_gcpy-<ver>.tar.gz
sha256sum /tmp/sdist.tar.gz
tar tzf /tmp/sdist.tar.gz | grep -i license     # license_file must be packaged
```

Then diff the declared requirements between the old and new version to
see whether the recipe's dependency lists need to change at all:

```bash
for v in <old> <new>; do curl -s https://pypi.org/pypi/geoschem-gcpy/$v/json \
  | python3 -c "import json,sys;print('\n'.join(sorted(json.load(sys.stdin)['info']['requires_dist'] or [])))" > /tmp/req_$v.txt; done
diff -u /tmp/req_<old>.txt /tmp/req_<new>.txt
```

Note that gcpy declares **exact `==` pins** for its whole stack, including
non-Python entries (`esmf`, `tk`, `netcdf-fortran`) and even
`python==3.13`. These are deliberately *not* mirrored into the recipe, and
docs/test tooling (`ipython`, `jupyter`, `pytest`, `sphinx*`) is
deliberately excluded from `run`. Keep that convention; a new `==` entry
upstream is not by itself a reason to add a run dependency.

**Selectors.** A `py<NNN` skip is dead code when conda-forge's global
`python_min` is already at or above `NNN` — the selector can never match.
Prefer `skip: true  # [win]` plus the real constraint expressed where it
belongs (a dependency's own `python >=` floor).

**`--no-deps` is redundant here.** conda-build's
`_set_env_variables_for_build()` already sets `PIP_NO_DEPENDENCIES`,
`PIP_IGNORE_INSTALLED` and `PIP_NO_INDEX` for every build script, so pip
never consults gcpy's `==` pins regardless. Adding it to the pip line is
fine — sibling feedstocks (`xesmf`, `cf_xarray`) use
`--no-deps --no-build-isolation` and it documents intent — but do not
justify it as a fix for a real failure, because it prevents nothing.

**Never hand-edit** `.ci_support/`, `.azure-pipelines/`,
`.github/workflows/`, `azure-pipelines.yml` or `README.md`. They are
conda-smithy output and encode the current global pinning, which cannot be
reconstructed by hand.

## 6. Open the PR, then let the bot rerender

Prefer the bot over a local `conda-smithy rerender`: it renders with the
pinning that is current at that moment, which is exactly what reviewers
and CI will use. Post it as its own comment:

```
@conda-forge-admin, please rerender
```

Two traps worth knowing:

- **Never write `@conda-forge-admin` in the PR body as prose.** The
  webservice re-parses the body on *every* edit; a mention with no command
  after it produces a "I couldn't find any valid commands in the request"
  reply. That message is harmless and says nothing about whether the
  rerender worked — check the commit list and timestamps before reacting
  to it.
- `gh pr edit` can abort on the Projects-classic GraphQL deprecation
  without changing anything. Patch via REST instead, and verify:

  ```bash
  gh api -X PATCH repos/conda-forge/geoschem-gcpy-feedstock/pulls/<N> -F body=@body.md
  ```

Then confirm the rerender did what you expected — new variant files added,
retired interpreters removed, any now-global migration file deleted:

```bash
git fetch origin <topic-branch>
git diff --name-status HEAD origin/<topic-branch>
```

Also watch for a competing `regro-cf-autotick-bot` version PR. It branches
off `main`, so it will carry none of your pin or selector work; close it
referencing yours rather than letting both run.

## 7. Verify on the channel, not just in CI

Green CI only proves the variants build. Confirm the published matrix is
complete and correctly pinned, then install and import for real:

```bash
conda search -c conda-forge --override-channels 'geoschem-gcpy=<ver>' --info | grep -E 'esmf|xesmf|python '
conda create -n gcpy<ver>-py<NNN> -c conda-forge python=<N.NN> geoschem-gcpy=<ver>
conda run -n gcpy<ver>-py<NNN> python -c "import gcpy; print(gcpy.__version__)"
```

Check both subdirs (`linux-64` and `osx-64`) — `conda search` reports only
the current platform unless given `--platform`.

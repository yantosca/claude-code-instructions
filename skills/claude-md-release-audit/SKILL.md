---
name: claude-md-release-audit
description: Re-verify a repository's CLAUDE.md (or AGENTS.md, or any agent-instruction file) against the code after a version release, finding wrong file paths, wrong CMake/build option names, stale version floors, claims that actually belong to a superproject or sibling submodule, and new release files the instructions never mention. Use when asked to audit, check, update, or scan CLAUDE.md against a repository, to verify agent instructions after a release, or to review CLAUDE.md for a new GEOS-Chem, GCHP, HEMCO, or GCPy version.
---

# Auditing CLAUDE.md against the code after a release

Method for checking whether a repository's agent-instruction file — `CLAUDE.md`,
`AGENTS.md`, or similar — still matches the tree it describes, and for extending
it to cover what a release just added. This is a companion to the
`docs-vs-code-validation` skill: that one starts from the changelog and walks a
documentation *tree* (toctrees, autodoc, sibling-page drift, a doc rebuild to
verify). This one has a different target and different failure modes — a single
file, no build step, and errors graded by how badly they misdirect an agent
rather than by how they confuse a reader.

Why it needs its own pass: `CLAUDE.md` is loaded into every session's context
and read as authoritative, but nothing validates it and no test fails when it
goes stale. A wrong path in prose is a nuisance; a wrong path in `CLAUDE.md`
gets acted on.

## 1. Grade findings by blast radius, not by section order

Sort what you find, because the fix order matters:

1. **Wrong paths and wrong identifiers** — worst. An agent opens the file, gets
   nothing, and may then "helpfully" create it or edit the wrong sibling.
2. **Wrong option/switch/flag names** — a command that fails confusingly, or
   silently does nothing.
3. **Plausible-but-wrong version floors and defaults** — acted on without
   suspicion precisely because they look specific.
4. **Over-general universals** ("each X has a Y") — send an agent looking for
   something that exists in only some cases.
5. **Omissions** — mildest; the agent falls back on reading the code.

## 2. The dominant failure mode is superproject/submodule misattribution

When the repo is a submodule that is never built alone, claims drift upward
into the wrapper and downward from siblings, and both read as if they were about
"this repo."

Before believing any claim about what the repo contains or builds:

- **Resolve every symlink**, at both levels. `ls -la` the superproject root.
  In one audit `CLAUDE.md` said the wrapper symlinks `run/`, `test/` and
  `spack/` "from this repo" — `spack` was real, but it pointed into a
  *different* submodule's docs tree. And `run` pointed at
  `src/GEOS-Chem/run/GCClassic/`, one level deeper than stated.
- **Read the superproject's `.gitmodules`** and list its `src/`. Sibling
  submodules are the thing most often missing entirely, and the most valuable
  to add: "emissions live in HEMCO, photolysis in Cloud-J, aerosol
  thermodynamics in HETP, all separate repos" prevents an agent editing the
  wrong repository, which no amount of in-repo detail will.
- **Check where a build target comes from.** If the repo links targets it never
  defines, say so — it is the clearest possible evidence for "this cannot be
  configured standalone." Likewise a root `CMakeLists.txt` with no `project()`
  that nonetheless prints `${PROJECT_VERSION}`.

## 3. Verify universals; never generalize from one instance

Statements of the form "each `<dir>/<impl>/` has a `<script>`" are cheap to
write and frequently false. Grep the whole set and count. Two real examples
from one file: "run-directory creation scripts for each implementation" held for
2 of 6 directories; "each subdirectory is a generated solver" held for 4 of 7
(one was a dormant, commented-out mechanism with no generated files, another was
a stub directory that is not a mechanism at all).

```bash
ls run/*/createRunDir.sh          # the claim, tested in one line
```

Where a set is genuinely non-uniform, a small table of "what each one actually
is" beats a sentence that averages over them.

## 4. Options and switches: find where they are *declared*, not just consumed

Grepping `${FOO}` finds the consumer, which tells you the switch exists but not
its name, default, or owner. Grep for `option(` and `set(... CACHE ...)`.

Three traps, all encountered in one pass:

- **The declaration is in another repo.** The audited repo contained no
  `option()` at all; every user-facing switch was declared in the
  superproject's configure script. Saying so is the useful fact.
- **A component is gated by a switch that does not share its name.** A
  `GeosRad/` directory was gated by `RRTMG`; there was no `GeosRad` switch.
- **A component has no switch, or no build integration at all.** One "optional
  component" was linked unconditionally (always compiled in); another had no
  `CMakeLists.txt` anywhere and was driven by its own shell scripts. A blanket
  "these are enabled by their own CMake switches" was wrong for three of the
  five it named.

Also check whether an option is *validated*. An unvalidated `MECH`/`MODE`-style
string that fails later on a missing target, rather than with a clear error, is
worth a warning line in `CLAUDE.md`.

Cross-check the switch names against the repo's own test harness if it has one —
the function that assembles configure flags for the test matrix is usually the
most reliable list in the tree.

## 5. Re-read every CLI usage block; do not trust remembered flags

Prefer the script's own `usage=` string and its `getopt`/argparse spec over its
header comment *and* over its README. In one audit all three disagreed for the
same script: the header documented a `-n` the script rejects, the README omitted
`--quick` and mangled the long-form names, and a vestigial `s:` in the getopt
spec had no case branch (so `-s foo` hangs). Quote from the spec, not the prose.

While there, capture the *preconditions* a flag has, not just its name — these
are what actually block a run: a mode that only works on recognized compute
sites, a script that refuses to run inside the source tree, an environment guard
that trips on any active conda env rather than only one with the offending
library. Narrow an over-broad claim here too ("all test scripts do X" was true
of 4 of ~12).

## 6. For a dependency version floor, find the machine-readable declaration

"Requires KPP 3.4.0+" conflates four things that drift apart:

- where the floor is **declared** in a form the tooling reads;
- what **enforces** it (which may be the upstream tool, not the local script);
- what the changelog **says** the minimum is;
- what the checked-in **artifacts were generated with**.

**Prefer a declaration the build actually consumes over any prose.** A
`#MINVERSION` directive, a `>=` pin in a package manifest, a
`cmake_minimum_required`, a version guard in a header — these cannot go stale
without breaking something, so they outrank both the changelog and the script
comments. Grep for the directive across every variant that declares it:

```bash
grep -rn "MINVERSION" KPP/*/*.kpp     # authoritative; one line per mechanism
```

Two traps, both hit in a single audit of one release:

- **The changelog contradicted itself within one release section.** Two bullets
  said the minimum went "from 3.2.0 to 3.4.0" and that solvers were
  "regenerated with KPP 3.4.0"; a later bullet in the *same* section said
  `#MINVERSION` was changed to 3.5.0, and every `.kpp` file plus every
  generated-solver header said 3.5.0. Taking the newest mention inside the
  current release's section is not enough — reconcile the prose against the
  declaration, and trust the declaration.
- **The floor was enforced upstream, not locally.** The wrapper build script did
  no version check at all and its header comment named a version six releases
  old, which reads as "nothing enforces this." The generator itself enforced
  `#MINVERSION`. Say *what* produces the error, so nobody documents a check that
  does not exist or an absence that is not real.

While you have the declarations side by side, diff them against each other —
this is where a release's stragglers show up. Here three mechanisms declared
3.5.0 and a fourth still declared 3.4.0.

## 7. Diff the release against the version the file was written for

```bash
git diff <prev-tag>..HEAD --stat
git log --oneline <prev-tag>..HEAD
```

Then check every new or renamed top-level file for a `CLAUDE.md` mention. A
release is exactly when governance and tooling files appear — a PR template, a
`GOVERNANCE.md`, a `SECURITY.md`, a `.gitattributes`, a version-bump script —
and an agent-instruction file written before them silently omits all of it.

Two that specifically earn a place in `CLAUDE.md`, because they change what the
agent should *do*:

- **An AI-disclosure section in the PR template.** If the project asks
  contributors to disclose AI assistance, the agent preparing the PR needs to
  know.
- **`.gitattributes` line-ending policy.** `eol=lf` enforcement means never
  writing CRLF into scripts or source.

## 8. Say plainly when there is no CI

An agent that assumes a PR gate exists will wait for one, or will treat "tests
pass" as someone else's problem. If the only workflow is a stale-bot on a
schedule, state that nothing gates a PR automatically and name what does serve
as verification (the repo's own test drivers, maintainer benchmark runs). Absence
of CI is a fact worth documenting, not a gap to leave silent.

## 9. Fan out by section, with explicit TRUE/FALSE per claim

`CLAUDE.md` is short enough to enumerate exhaustively, so convert it into a
numbered list of its own assertions and hand each subagent a slice — layout
claims, build/CMake claims, test-infrastructure claims — with the instruction to
report TRUE or FALSE **per item, with quoted evidence**. This beats "check
whether CLAUDE.md is accurate," which returns impressions.

Give each agent the file paths to read, and require it to quote the line it is
judging from. Then apply the `docs-vs-code-validation` rule about re-verifying
findings yourself: a subagent reporting a version floor from the changelog may
well have read the wrong release's section (see step 6 — this happened).

## 10. Version-bump helpers rarely cover every version string

If the repo has a release script, read it and enumerate what it touches, then
after a bump:

```bash
git grep -n "14\.8\.0"      # the old version, after bumping
```

Classify every hit into three groups, because they have different rules, and
record the classification in `CLAUDE.md` so the next release does not re-derive it:

- **Script-handled** — the changelogs, the citation metadata.
- **Manual headers** — e.g. a `Version:` line inside a mechanism definition
  file, which no script updates.
- **Data paths** — references to a versioned directory of input/restart files on
  a data portal. These should change *only* when new files for that version
  actually exist, not on every bump.

Two failure modes to check for by name:

- **Version metadata that silently went stale.** A `CITATION.cff` still naming
  the *previous* release after the current one shipped — and carrying a
  `date-released` that disagreed with the changelog's date for that same
  version. Nothing validates these files, they are not part of any build, and
  the error is invisible until someone cites the software. Check them against
  the changelog on every release audit, and note that a bump script which stamps
  `date(1)` writes *today's* date, not the release date.
- **A substitution that no-oped.** `sed -i` exits 0 when its pattern matches
  nothing, so a script can report success having changed nothing. In one case
  the `[Unreleased]` heading the script keys on did not exist in one of the two
  changelogs it targets. Confirm each intended edit actually landed rather than
  trusting the script's own output.

## 11. Re-verify the edited file mechanically

Every backticked path in `CLAUDE.md` is a testable claim, which makes the cheap
regression test possible — extract and resolve them all:

```bash
grep -o '`[A-Za-z_.][A-Za-z0-9_./-]*`' CLAUDE.md | tr -d '`' | grep '/' |
  while read -r p; do [ -e "$p" ] || echo "MISSING: $p"; done
```

Expect a few deliberate non-hits and eyeball rather than trust the list: paths
relative to a subdirectory the surrounding table establishes, upstream repo
slugs (`org/repo`), superproject-relative paths, and any path you cite
*precisely because* it is wrong elsewhere. Run the bare-filename half with
`find -name` to confirm each file is where the text says it is — that is the
check that catches an off-by-one-directory path, the single most common error.

Finally, confirm the fixed file does not re-introduce an error it inherited.
A wrong path in `CLAUDE.md` often came from a wrong path in the changelog; fix
the instruction file, and mention the discrepancy rather than silently
propagating the released changelog's version.

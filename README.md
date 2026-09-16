# claude-code-instructions

Reusable [Claude Code](https://claude.com/claude-code) working methods for the GEOS-Chem
model ecosystem — GEOS-Chem Classic, GCHP, and GCPy.

Packaged as a Claude Code plugin so it can be installed on any machine and shared with the
GEOS-Chem Support Team.

## Skills

| Skill | What it covers |
| --- | --- |
| `geos-chem-benchmark-review` | Review a GCClassic/GCHP version-comparison benchmark and trace a difference back to the PR that caused it |
| `geos-chem-newsletter-outline` | Draft the outline for the next GEOS-Chem newsletter issue |
| `geos-chem-publications-lookup` | Pull current GEOS-Chem publications from Google Scholar and resolve DOIs via Crossref |
| `gcpy-add-python-version` | Add a new Python version to GCPy, through to the conda-forge feedstock |
| `gcpy-feedstock-pr` | Land a PR on the `geoschem-gcpy` conda-forge feedstock and verify it on the channel |
| `openmp-collapse-schedule-audit` | Audit Fortran/OpenMP loops for `COLLAPSE(n)` and `SCHEDULE` fit |
| `sphinx-docs-validation` | Audit a Sphinx/ReadTheDocs tree for build warnings and hygiene issues |
| `docs-vs-code-validation` | Check whether docs and CHANGELOG still match what the code does |

## Install

```bash
/plugin marketplace add yantosca/claude-code-instructions
/plugin install geos-chem-methods
```

Update with `/plugin marketplace update geos-chem-methods`, which pulls the latest
`main` from GitHub.

To hack on the skills themselves, add the marketplace from a local clone instead —
edits then take effect without a push:

```bash
/plugin marketplace add ~/repos/claude-code-instructions
/plugin install geos-chem-methods
```

## Always-on rules

`Git-commit-message-style.md` is not a skill — it applies to every commit, so it belongs in
context permanently rather than waiting on a trigger. Wire it in once per machine:

```bash
mkdir -p ~/.claude
echo '@~/repos/claude-code-instructions/Git-commit-message-style.md' >> ~/.claude/CLAUDE.md
```

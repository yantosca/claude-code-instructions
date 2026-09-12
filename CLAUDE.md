# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

A personal/GCST repository of reusable Claude Code working methods for the GEOS-Chem
ecosystem. It contains no source code, build system, or tests — every file is Markdown or
JSON manifest.

The repository is packaged as a **Claude Code plugin** (and is its own single-plugin
marketplace), so the methods can be installed on any machine and shared with the GEOS-Chem
Support Team rather than living in one checkout.

## Layout

- `skills/<name>/SKILL.md` — one skill per working method. Each carries YAML frontmatter with
  a `name` (which **must** equal the directory name) and a `description`.
- `.claude-plugin/plugin.json` — plugin manifest.
- `.claude-plugin/marketplace.json` — marketplace manifest; `source` is `./` because the
  repository root *is* the plugin.
- `Git-commit-message-style.md` — deliberately **not** a skill. It is an always-on rule, wired
  into `~/.claude/CLAUDE.md` via an `@` import so it is in force for every commit rather than
  waiting on a skill trigger.
- `CODE_OF_CONDUCT.md`, `SECURITY.md` — repository governance, not instructions to Claude.

## Adding a new method

1. Create `skills/<kebab-case-name>/SKILL.md`.
2. Give it frontmatter where `name` matches the directory exactly.
3. **Spend the effort on `description`.** A skill loads only when its description matches how
   the task is actually phrased, so the description must carry the trigger vocabulary — tool
   names, error strings, model/version names, the words someone would really type. A skill
   with a vague description silently never fires, which is worse than not having it.
4. Document the *method*, not a snapshot of one run's results. Every skill here says so in its
   own opening paragraph; keep that convention.
5. Cross-reference sibling methods by skill name (e.g. "the `docs-vs-code-validation` skill"),
   not by file path — paths change, and a broken path is invisible at runtime.

## Conventions

- Keep line endings LF and files plain text (enforced via `.gitattributes`).
- Commit messages follow `Git-commit-message-style.md`.

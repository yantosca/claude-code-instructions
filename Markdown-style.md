# Markdown Style

My preferred format for Markdown I will paste into GitHub or commit to a repository. It applies to every repository, like `Git-commit-message-style.md`.

## Rule

**Never hard-wrap a paragraph or list item.** Write each paragraph, list item, and blockquote line as one unbroken line, however long, and put line breaks only between blocks: paragraphs, list items, headings, tables, and code fences.

GitHub renders a single newline in a PR description, issue, or comment as a hard line break, so text wrapped at 80 or 100 columns comes out ragged once pasted. Committed `.md` files render correctly either way, but they get copied into PRs and comments, so they follow the same rule.

## What keeps its own line structure

- Fenced code blocks, including shell commands and YAML.
- Tables, one row per line.
- Nested list items, each starting on its own line with its indent.

## What this rule does not cover

- **Git commit messages.** These stay wrapped as described in `Git-commit-message-style.md`, because `git log` does not reflow text.
- **Comments inside source code and YAML files.** Wrap these to the surrounding code's width.

## When editing an existing file

If a file is already hard-wrapped, write new text without wraps. Reflow the whole file only when asked, and check afterwards that only whitespace changed, since a line ending in `/` or `-` must be joined without a space.

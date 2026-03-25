# Agents Repository

Portable instruction library for AI agents, served as raw markdown via GitHub.

## Structure

- `AGENTS.md` — Index file. Entry point for agents consuming this repo via `raw.githubusercontent.com`.
- `skills/` — One markdown file per skill. Plain markdown, no YAML frontmatter.

## Rules

- **AGENTS.md is the index**: When a skill is added, edited, or removed, update `AGENTS.md` to reflect the change (add/update/remove the table row).
- **One file per skill**: Each skill lives at `skills/<skill-name>.md`.
- **Plain markdown only**: No YAML frontmatter, no tool-specific syntax. Files must be useful to any agent that can read markdown.
- **Self-contained**: Each skill file must contain all instructions needed to execute the skill — no cross-references to other skills unless explicitly needed.

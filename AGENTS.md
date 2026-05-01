# Agents

A collection of reusable instructions for AI agents. Each file is a self-contained markdown document that any agent can read and follow.

## How to use

If you are reading this from `raw.githubusercontent.com`, fetch any file listed below by requesting:

```
https://raw.githubusercontent.com/juanlb/agents/main/<path>
```

## Skills

Reusable task instructions located in the `skills/` folder.

| Skill | Description | Path |
|-------|-------------|------|
| show-proof | Create a live proof-of-work document that verifies completed work through observable behavior (before/action/after). Uses `showboat` for document assembly, `rodney` for browser automation, and `mmdc` for Mermaid diagrams. | `skills/show-proof.md` |
| explain | Create a graphics-heavy document that explains a situation — architecture, system, bug, or flow — using `showboat`, `rodney`, and Mermaid diagrams. All resources are written under `/tmp`, never the working directory. | `skills/explain.md` |

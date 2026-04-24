# CLAUDE.md / AGENTS.md

Portable, cross-machine agent configuration. This repo is the source of truth for the guidance files that Claude Code, GPT Codex, and other coding agents auto-load when they start in a workspace.

## Contents

- `CLAUDE.md` — read by Claude Code (Anthropic)
- `AGENTS.md` — read by GPT Codex and other AGENTS.md-compatible agents (Cursor, Aider, Gemini CLI)
- `knowledge/` — markdown summaries and design notes indexed by the guidance files
- `papers/` — source PDFs referenced from `knowledge/`

`CLAUDE.md` and `AGENTS.md` describe the same workspace conventions (layout, knowledge-base pattern, environment policy). They differ only in the agent name, so each agent sees itself addressed directly.

## Design principles

- **Workspace-abstract.** The guidance files describe a pattern (subprojects, knowledge-base index, environment policy), not one specific project. Drop this repo into any workspace and the conventions apply.
- **Subproject autonomy.** Individual subprojects keep their own `CLAUDE.md` / `README.md` for project-specific commands and architecture. The guidance here stays at the workspace layer.
- **No system-level installs.** Every subproject uses `uv` for environment isolation. Agents are instructed never to touch the host Python, `apt`, or CUDA.
- **Knowledge base is an index, not a copy.** Long-form notes live as their own markdown files in `knowledge/`; `CLAUDE.md` / `AGENTS.md` only point at them.

## Usage

```bash
git clone git@github.com:elichen3051/CLAUDE.md.git agent-sync
```

Start Claude Code or Codex from the cloned directory (or any directory whose parent chain contains it). The agent auto-loads the guidance and the knowledge-base index on session start.

## Adding a new reference

1. Put the source PDF in `papers/` with its upstream filename (arXiv id or conference-provided name). Do not rename.
2. Write a summary markdown in `knowledge/` with a descriptive filename (`<topic>_<what_it_documents>.md`). If derived from a paper, add a `Source: papers/<filename>` line at the top.
3. Register the topic + relative path as one row in the KB index table inside **both** `CLAUDE.md` and `AGENTS.md`.

## Keeping the two guidance files in sync

Because `CLAUDE.md` and `AGENTS.md` are separate files (not symlinked), edits to one must be mirrored to the other. If the maintenance burden grows, switch to a symlink:

```bash
rm AGENTS.md && ln -s CLAUDE.md AGENTS.md
```

Both Claude Code and Codex follow symlinks transparently.

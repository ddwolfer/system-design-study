---
name: kg-init
description: Wire Multi-knowledgeGraph into THIS project — generates .mcp.json / .claude/settings.json / .codex/config.toml / .gemini/settings.json with the right paths and optional briefing blocks. Use when the user wants to set up the knowledge graph for a project they cloned Multi-knowledgeGraph into.
---

You are wiring the Multi-knowledgeGraph repo into the current project. Do NOT re-implement what `scripts/setup-project.js` already does — your job is to collect a few choices, find the KG repo, build one command, and run it.

## 1. Locate the Multi-knowledgeGraph clone

The KG repo is usually a subdirectory of the user's project. Try in order:

1. Look for a directory under cwd containing both `main.js` AND `lib/embeddings.js` AND `hooks/session-start.js`. Common names: `kg`, `multi-knowledgeGraph`, `multi-knowledge-graph`, `knowledgeGraph`.
2. If unique match → use it.
3. If multiple matches → ask the user which one.
4. If no match → ask the user where they cloned it (or tell them to clone first).

Treat that path as `KG_DIR` for the rest of this skill.

## 2. Collect (single AskUserQuestion call, 4 questions)

Use AskUserQuestion to ask all four at once. Defaults marked **(Recommended)** as the first option.

1. **DB setup?**
   - single (just `knowledge.db`) — **(Recommended)**
   - preset (main + research + scratch)
   - custom (you'll provide names)
2. **Which platforms?** (multiselect)
   - claude — **(Recommended)** default-on
   - codex
   - gemini
3. **Inject briefing block** into CLAUDE.md / AGENTS.md / GEMINI.md?
   - yes — **(Recommended)**
   - no
4. **KG directory** — only ask if step 1 found multiple candidates or none. Skip if uniquely detected.

If user picked **custom** DB, follow up: "Custom DB names (comma-separated)?" Default: `main,research`.

## 3. Run the engine

Build one command and run it via Bash. Map answers to flags:

| Answer | Flag |
|--------|------|
| db: single / preset | `--db single` or `--db preset` |
| db: custom + names | `--db custom --custom-dbs <csv>` |
| platforms | `--platforms <csv>` |
| briefing: no | `--no-briefing` |

```bash
node <KG_DIR>/scripts/setup-project.js --db <mode> --platforms <csv> [--custom-dbs <csv>] [--no-briefing]
```

Don't add `--interactive` — you've already collected answers.
Don't add `--reset-git` unless the user explicitly asked for fresh git history.

## 4. After it runs

- Relay the engine's "Next steps" output to the user verbatim — it tells them to restart Claude Code, trust the project for Codex, etc.
- If it exits non-zero: report the failing step. The fix is always "resolve the cause, then re-run this skill" — every step is idempotent.
- **Do not hand-edit** `.mcp.json` / `.claude/settings.json` / `.codex/config.toml` / `.gemini/settings.json` / `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`. The engine owns them via markers. Re-running this skill is always the right answer.

# Current Task

> This file describes what Claude Code should be working on right now. It is overwritten by claude.ai as work progresses.

## Status: Not started — awaiting first session

## Task: Foundation setup

Your first task is to set up the foundation of the project. Do not start writing application logic yet. The goal is to have a clean, runnable Next.js project skeleton with the right structure and tooling in place, before we add the real features.

### Steps

1. **Read BRIEFING.md first.** Make sure you understand the project context, architecture, and tech stack decisions.

2. **Initialize the private repository** at `github.com/stabmegane/bozas-quick-entry` (currently empty). Clone it locally on the codebox alongside this bridge repo.

3. **Create a Next.js 14+ project** with the App Router:
   - TypeScript
   - Tailwind CSS
   - ESLint
   - Default import alias `@/*`

4. **Set up the project structure** with these folders:

```
src/
  app/                    Next.js routes
  components/             React components
  lib/
    config/               Config loader + types
    pylon/                Pylon API client (login, getdata, postdata)
    parser/               Voice command parsers (regex now, llm later)
    db/                   SQLite audit log
  types/                  Shared TypeScript types
config/
  config.example.json     Template config (no real secrets)
scripts/
  setup.ts                CLI setup wizard
  test-config.ts         Connection diagnostic tool
```

5. **Create the config loader** in `src/lib/config/`:
   - Define TypeScript types matching the config.json schema in BRIEFING.md
   - Read config from path specified by `CONFIG_PATH` env var, default `./config/config.json`
   - Validate required fields exist on load; fail fast with clear error message
   - Export a singleton `config` object

6. **Create `config/config.example.json`** matching the schema in BRIEFING.md with placeholder/example values (NOT real credentials).

7. **Create a `.gitignore`** that excludes:
   - `node_modules/`
   - `.next/`
   - `config/config.json` (real config never committed)
   - `data/` (SQLite files never committed)
   - `.env*`
   - OS files (.DS_Store, Thumbs.db)

8. **Create a minimal README.md** for the private repo with:
   - Project description (one paragraph)
   - Quick start: clone, install, copy config.example.json to config.json, edit, run
   - Link to this bridge repo for full context

9. **Verify it runs:** `npm run dev` should start the Next.js dev server on port 3000, showing the default starter page. We will replace this page in the next task.

10. **Commit and push** to the private repo with message: `Initial project skeleton`

### Constraints

- Do not write any voice recognition, NLP, Pylon API client, or UI feature logic yet. That comes in later tasks.
- Do not commit any real credentials, API codes, or customer data.
- Use exact column names from BRIEFING.md when defining types — do not invent variations.
- If anything is ambiguous, write the question to `questions.md` in this bridge repo and stop. Do not guess.

### When done

Update this file's Status to `Completed` and add a short summary of what you did and what files were created. Then ask the owner what to work on next.

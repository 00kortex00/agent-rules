---
name: init-project-rules
description: Scaffolds or integrates AI agent rules into a project. Clones rules from the agent-rules repository, generates CLAUDE.md/AGENTS.md tailored to the stack. For new projects, it interviews the user. For existing projects, it analyzes the stack, adds the rules, and proposes a step-by-step refactoring plan to align the codebase with the chosen architectural rules using a TODO.md checklist. Trigger when the user wants to start a new project, format an existing project, or initialize rules.
---

# Project Initialization Skill

This skill sets up agent rules for both **new** and **existing** projects. It selects relevant rules from `github.com/00kortex00/agent-rules`, clones them into `.agents/rules/`, generates IDE integration files, and (for existing projects) audits the codebase to propose a refactoring plan.

## Overview of steps

1. Analyze environment & Interview
2. Rule selection
3. Clone rules
4. Generate `project.md`
5. Generate `CLAUDE.md`, `AGENTS.md`, and symlinks
6. Existing Project Audit & Refactoring (if applicable)

---

## Step 1 — Analyze environment & Interview

First, determine if the project is new (empty directory) or existing (has `package.json`, source files, etc.).

**If it's an EXISTING project:**
1. Silently analyze the current stack by checking `package.json`, `tsconfig.json`, and the file tree (e.g., detecting Next.js, Vite, Elysia, Drizzle).
2. Report the detected stack to the user and ask for confirmation.
3. If the architecture is mixed or unstructured (e.g., flat `components/` folder without UI/feature separation), ask them to choose a target architectural pattern from the rules (e.g., "Do you want to migrate to Atomic Design or FSD?").
4. Also ask: "Are there any custom constraints or conventions the agent should know?"

**If it's a NEW project:**
Ask the user the following. You can ask all at once in a conversational message — don't send them one by one:

**Required:**
- What is this project? (brief description — 1-2 sentences)
- Frontend, backend, or both?
- Is this a monorepo (multiple apps) or a single app?

**Based on answers, ask follow-up:**

If frontend:
- Which framework? (default: Next.js App Router — mention this)
- Which architecture pattern? Atomic Design or FSD?
- UI library? (default: HeroUI — mention this, alternatives: shadcn/ui, none)

If backend:
- Which framework? (default: Elysia — mention this)
- Database? (default: Drizzle + PostgreSQL — mention this)

**Always ask:**
- Anything custom about this project that the agent should know? (team conventions, specific constraints, etc.)

Present defaults clearly — the user can just confirm them or pick something else. Keep it conversational, not a form.

---

## Step 2 — Rule selection

Based on answers, build the list of rule files to clone.

Always include:
```
general/code-style.md
general/git.md
general/security.md
general/testing.md
general/tooling.md
general/scaffolding.md
general/workflow.md
```

Conditionally include:

| Condition | Add |
|---|---|
| monorepo | `general/monorepo.md` |
| has frontend | `frontend/react-components.md`, `frontend/react-state.md` |
| frontend = Next.js | `frontend/nextjs.md` |
| has backend | `backend/structure.md`, `backend/api-design.md` |
| backend = Elysia | `backend/elysia.md` |
| database = Drizzle | `backend/database-drizzle.md` |

---

## Step 3 — Clone rules

Run sparse checkout to clone only the selected files into `.agents/rules/`:

```sh
# 1. Create temp dir and clone with no blobs
git clone --filter=blob:none --sparse https://github.com/00kortex00/agent-rules .agents/_tmp
cd .agents/_tmp

# 2. Set sparse checkout to selected files
git sparse-checkout set <file1> <file2> ...

# 3. Copy files preserving folder structure into .agents/rules/
cp -r general ../rules/
cp -r frontend ../rules/   # if applicable
cp -r backend ../rules/    # if applicable

# 4. Clean up temp dir
cd ..
rm -rf _tmp
```

Build the `git sparse-checkout set` arguments dynamically from the selected rule list.

---

## Step 4 — Generate project.md

Create `.agents/rules/project.md` describing this specific project. This file is always custom — never cloned from the repo.

```markdown
---
id: project
category: project
description: Project-specific context and conventions
---

# Project: <name>

## Description

<1-2 sentence description from the user>

## Stack

<list the confirmed stack — framework, database, UI library, etc.>

## Architecture

<frontend pattern if applicable: Atomic Design or FSD>
<monorepo structure if applicable>

## Custom conventions

<anything extra the user mentioned, or "None." if nothing>
```

---

## Step 5 — Generate CLAUDE.md and AGENTS.md

### CLAUDE.md

```markdown
# CLAUDE.md

@.agents/rules/project.md
@.agents/rules/general/code-style.md
@.agents/rules/general/git.md
@.agents/rules/general/security.md
@.agents/rules/general/testing.md
@.agents/rules/general/tooling.md
@.agents/rules/general/scaffolding.md
<additional rules based on selection>
```

Add only the rules that were actually cloned. No glob — list each file explicitly so the agent knows exactly what's included.

### AGENTS.md and IDE rules

`AGENTS.md` has the same content as `CLAUDE.md` — it targets other agents (OpenAI Codex, Gemini, etc.) that read this file instead.

To ensure compatibility with IDE-based agents like Cursor and Windsurf, create symlinks pointing to `CLAUDE.md`:

```sh
ln -s CLAUDE.md .cursorrules
ln -s CLAUDE.md .windsurfrules
```

---

## Final output

After completing all steps, confirm to the user:

```
✅ Cloned X rule files into .agents/rules/
✅ Created .agents/rules/project.md
✅ Created CLAUDE.md
✅ Created AGENTS.md (and symlinks: .cursorrules, .windsurfrules)

Rules included:
- general: code-style, git, security, testing, tooling, scaffolding, workflow<, monorepo>
- frontend: react-components, react-state<, nextjs>  (if applicable)
- backend: structure, api-design, elysia, drizzle (if applicable)
```

Remind the user: if they add new tools or change the stack later, they can re-run this skill to update the rules.

---

## Step 6 — Existing Project Audit & Refactoring

If you are running this skill in an **existing project**, after generating the rules:
1. Analyze the current codebase against the newly cloned rules.
2. Identify discrepancies (e.g., lack of `import type`, unstructured `components/` directory, magic numbers, missing `docs/TODO.md`).
3. Prepare a short audit report and propose a refactoring plan to align the project with the established standards.
4. **Important:** If the user approves the refactoring, create a `docs/TODO.md` file with a step-by-step checklist.
5. Include dedicated tasks in the `TODO.md` to **analyze the entire project and write initial documentation** in the `docs/` folder (e.g., `docs/architecture.md`, `docs/state.md`) to establish the "Lazy Context" baseline.
6. Execute the refactoring incrementally (one step at a time), checking off items in the `TODO.md` to conform strictly to the newly adopted `general/workflow.md`.

---
name: init-project
description: Scaffolds or integrates AI agent rules into a project. Clones rules from the agent-rules repository, generates CLAUDE.md/AGENTS.md tailored to the stack. For new projects, it interviews the user. For existing projects, it analyzes the stack, adds the rules, and proposes a step-by-step refactoring plan. Can also update rules from GitHub and regenerate all IDE adapter files. Trigger when the user wants to start a new project, integrate rules into an existing project, or update existing rules.
---

# Project Initialization Skill

This skill operates in three modes:

- **New project** — interviews the user, selects rules, clones them, generates all IDE files from scratch.
- **Existing project** — detects the stack, integrates rules, audits the codebase, proposes a refactoring plan.
- **Update** — pulls the latest rules from GitHub, regenerates all IDE adapter files, keeps `project.md` untouched.

Rules are cloned from `github.com/00kortex00/agent-rules` into `.agents/rules/` — the single source of truth. All IDE-specific files are generated from this source.

## Supported tools and their generated files

| Tool | File(s) | How rules land |
|---|---|---|
| Claude CLI | `CLAUDE.md` | `@file` annotations only — never inline content |
| Copilot | `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md` | inlined content with `applyTo` frontmatter |
| Cursor | `.cursor/rules/*.mdc` | inlined content with `alwaysApply`/`globs` frontmatter |

## Mode detection (do this first)

Before anything else, determine which mode to run:

1. If the user explicitly says "update" or "update rules" → **Update mode** (jump to Step 7)
2. If the user explicitly says "audit" → **Audit mode** (jump to Step 8)
3. Else if `.agents/rules/` already exists:
   - Check whether the project is **fully initialized**: does `docs/` exist AND at least one IDE adapter file (`CLAUDE.md`, `.github/copilot-instructions.md`, or `.cursor/rules/`) exist?
   - If **fully initialized** → show the extended menu (see below) and wait for the user's choice
   - If **not fully initialized** (rules exist but docs/IDE files are missing) → ask: "Found existing rules but setup looks incomplete. Do you want to finish initialization or update rules from GitHub?"
4. Else if the directory has source files (`package.json`, etc.) → **Existing project mode**
5. Else → **New project mode**

### Extended menu (fully initialized project)

```
Found existing agent setup. What would you like to do?

1. Update rules       — pull latest rules from GitHub, regenerate all IDE adapter files
2. Audit project      — analyze codebase against current rules, find gaps, write/update docs/
3. Regenerate IDE files — re-generate CLAUDE.md / Copilot / Cursor files from .agents/rules/ (no GitHub fetch)
4. Re-initialize      — start fresh, re-interview stack and conventions
```

Wait for the user's choice:
- **1** → **Update mode** (jump to Step 7)
- **2** → **Audit mode** (jump to Step 8)
- **3** → Run **Step 5** only (skip clone and project.md generation)
- **4** → **New project mode** (Step 1 from scratch)

## Overview of steps

1. Analyze environment & Interview
2. Rule selection
3. Clone rules
4. Generate `project.md`
5. Generate IDE adapter files
6. Existing Project Audit & Refactoring (if applicable)
7. Update rules (Update mode only)
8. Audit mode (fully initialized project)

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
- Database? (default: Drizzle + PGLite for local / PostgreSQL for production — mention this; explain that PGLite runs in-process with no local Postgres needed, and automatically switches to real Postgres when `DATABASE_URL` is set)

**Always ask:**
- Should the agent **auto-commit** after each completed step? (default: no — mention this, explain it means one commit per TODO step, local only, no push)
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
| user said yes to auto-commit | `general/git-autocommit.md` |

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

## Step 5 — Generate IDE adapter files

`.agents/rules/` is the single source of truth. Every file below is generated from it. When inlining rule content, strip the YAML frontmatter block (the `---…---` header) from each file — only include the Markdown body.

---

### 5a. CLAUDE.md

`CLAUDE.md` must contain **only `@file` annotations** — one per rule. Do **not** inline any rule content here. Claude CLI expands these references natively, so the file stays short regardless of how many rules there are.

```markdown
# CLAUDE.md

@.agents/rules/project.md
@.agents/rules/general/code-style.md
@.agents/rules/general/git.md
@.agents/rules/general/security.md
@.agents/rules/general/testing.md
@.agents/rules/general/tooling.md
@.agents/rules/general/scaffolding.md
@.agents/rules/general/workflow.md
<additional rules based on selection>
```

List only the rules that were actually cloned. No globs. The file must never exceed ~20 lines.

---

### 5b. GitHub Copilot

Copilot reads `.github/copilot-instructions.md` as persistent context and auto-loads `.github/instructions/*.instructions.md` files based on `applyTo` patterns.

**`.github/copilot-instructions.md`** — project context summary:

```markdown
<body of .agents/rules/project.md>

Detailed coding rules are split into instruction files in `.github/instructions/`.
```

**`.github/instructions/<name>.instructions.md`** — one file per cloned rule (except `project.md`):

```markdown
---
applyTo: "**"
---

<body of the rule file>
```

For frontend or backend rules in a **monorepo**, use a more specific `applyTo` instead of `**`:

| Rule category | `applyTo` (monorepo) | `applyTo` (single app) |
|---|---|---|
| `frontend/*` | `"apps/web/**"` | `"**"` |
| `backend/*` | `"apps/api/**"` | `"**"` |
| `general/*` | `"**"` | `"**"` |

The `apps/web` and `apps/api` paths should match the actual workspace structure — adjust if the monorepo uses different names.

---

### 5c. Cursor

Create `.cursor/rules/<name>.mdc` for each cloned rule (except `project.md`):

```markdown
---
description: <value of `description` from the rule's frontmatter>
alwaysApply: true
---

<body of the rule file>
```

For frontend/backend rules in a **monorepo**, replace `alwaysApply: true` with a `globs` field:

```markdown
---
description: <description>
globs: ["apps/web/**"]   # or apps/api/** for backend rules
---

<body of the rule file>
```

---

## Final output

After completing all steps, confirm to the user:

```
✅ Cloned X rule files into .agents/rules/
✅ Created .agents/rules/project.md
✅ Generated IDE adapter files:
   - CLAUDE.md               (Claude CLI — @file annotations only)
   - .github/copilot-instructions.md + instructions/*.instructions.md  (Copilot)
   - .cursor/rules/*.mdc     (Cursor)

Rules included:
- general: code-style, git, security, testing, tooling, scaffolding, workflow<, monorepo>
- frontend: react-components, react-state<, nextjs>  (if applicable)
- backend: structure, api-design, elysia, drizzle (if applicable)
```

Remind the user: to get the latest rules from GitHub later, just say "update rules" — the skill will pull and regenerate everything automatically.

---

## Step 6 — Existing Project Audit & Refactoring

If you are running this skill in an **existing project**, after generating the rules:
1. Analyze the current codebase against the newly cloned rules.
2. Identify discrepancies (e.g., lack of `import type`, unstructured `components/` directory, magic numbers, missing `docs/TODO.md`).
3. Prepare a short audit report and propose a refactoring plan to align the project with the established standards.
4. **Important:** If the user approves the refactoring, create a `docs/TODO.md` file with a step-by-step checklist.
5. Include dedicated tasks in the `TODO.md` to **analyze the entire project and write initial documentation** in the `docs/` folder (e.g., `docs/architecture.md`, `docs/state.md`) to establish the "Lazy Context" baseline.
6. Execute the refactoring incrementally (one step at a time), checking off items in the `TODO.md` to conform strictly to the newly adopted `general/workflow.md`.

---

## Step 7 — Update rules (Update mode)

This mode refreshes rules from GitHub and regenerates all IDE adapter files. It does **not** touch `project.md` — project-specific context is always preserved.

### 7a. Read current rule list

Walk `.agents/rules/general/`, `.agents/rules/frontend/`, `.agents/rules/backend/` and collect all `.md` files, excluding `project.md`. This is the list of rules to re-fetch.

### 7b. Pull latest from GitHub

Re-run sparse checkout for exactly the same set of files:

```sh
git clone --filter=blob:none --sparse https://github.com/00kortex00/agent-rules .agents/_tmp
cd .agents/_tmp
git sparse-checkout set <file1> <file2> ...

cp -r general ../rules/
cp -r frontend ../rules/   # if present
cp -r backend ../rules/    # if present

cd ..
rm -rf _tmp
```

### 7c. Regenerate all IDE adapter files

With the updated rule files in place, regenerate every IDE adapter file following the same logic as Step 5 (5a through 5c). Overwrite the existing files completely — do not merge or diff.

`project.md` is never overwritten.

---

## Step 8 — Audit mode

This mode is for **fully initialized projects** where rules and IDE files are already in place. Goal: analyze the codebase against current rules, find gaps, and produce or update documentation.

### 8a. Load context

Read `.agents/rules/project.md` to understand the project's declared stack, architecture, and conventions. Collect the list of all rule files in `.agents/rules/`.

### 8b. Analyze the codebase

Silently scan the project against the loaded rules. Look for:
- Code style violations (import order, naming, `import type`, magic numbers, etc.)
- Architectural inconsistencies (e.g., feature logic in wrong layer, missing index files)
- Security concerns from `general/security.md`
- Missing or stale `docs/` files

### 8c. Check documentation coverage

Verify which `docs/` files exist and whether they reflect the current state of the project:

| Expected file | Purpose |
|---|---|
| `docs/architecture.md` | High-level structure, layers, data flow |
| `docs/state.md` | State management approach (if frontend) |
| `docs/api.md` | API overview (if backend) |
| `docs/TODO.md` | Active task checklist |

Note which files are missing or likely outdated.

### 8d. Present audit report

Show a short report:

```
## Audit Report

### Code quality gaps
- <issue 1 with file reference>
- <issue 2>
...

### Documentation gaps
- docs/architecture.md — missing
- docs/TODO.md — missing
...

### Proposed plan
1. <action 1>
2. <action 2>
...
```

Ask for approval before making any changes.

### 8e. Execute (after approval)

1. Create or update `docs/TODO.md` with the full checklist of proposed actions (mark items that are already done).
2. Write any missing `docs/` files based on codebase analysis.
3. Fix critical code issues if user confirms (otherwise leave them in `docs/TODO.md` for manual follow-up).
4. Always follow `general/workflow.md` — make incremental changes, not a big-bang rewrite.

### 7d. Confirm

```
✅ Updated X rule files in .agents/rules/ (project.md preserved)
✅ Regenerated IDE adapter files:
   - CLAUDE.md
   - .github/copilot-instructions.md + instructions/
   - .cursor/rules/
```

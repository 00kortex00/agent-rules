# agent-rules

A collection of reusable AI agent rules for TypeScript projects. Used by the project scaffolding skill to automatically configure new projects.

## Usage

### Automatic (via skill)

The project scaffolding skill clones the relevant rules into `.agents/rules/` and generates `CLAUDE.md` and `AGENTS.md` automatically. See skill documentation for details.

### Manual

Clone only the rules you need using sparse checkout:

```sh
git clone --filter=blob:none --sparse https://github.com/00kortex00/agent-rules .agents/rules
cd .agents/rules
git sparse-checkout set general frontend backend
```

Or fetch a single file:

```sh
mkdir -p .agents/rules/general
curl -o .agents/rules/general/code-style.md \
  https://raw.githubusercontent.com/00kortex00/agent-rules/main/general/code-style.md
```

---

## Structure

```
general/     # Language and tooling rules — apply to every project
frontend/    # React, state management, component architecture
backend/     # Elysia, Drizzle, API design, project structure
```

---

## Rules manifest

This manifest is used by the scaffolding skill to select and fetch rules.

```json
{
  "repo": "https://github.com/00kortex00/agent-rules",
  "raw": "https://raw.githubusercontent.com/00kortex00/agent-rules/main",
  "branch": "main",
  "rules": [
    {
      "id": "general-code-style",
      "file": "general/code-style.md",
      "category": "general",
      "tags": ["typescript", "code-style", "naming", "functions", "errors"],
      "always": true
    },
    {
      "id": "general-git",
      "file": "general/git.md",
      "category": "general",
      "tags": ["git", "commits", "branches"],
      "always": true
    },
    {
      "id": "general-security",
      "file": "general/security.md",
      "category": "general",
      "tags": ["security", "secrets", "validation", "dependencies"],
      "always": true
    },
    {
      "id": "general-testing",
      "file": "general/testing.md",
      "category": "general",
      "tags": ["testing", "bun", "unit", "integration"],
      "always": true
    },
    {
      "id": "general-tooling",
      "file": "general/tooling.md",
      "category": "general",
      "tags": ["bun", "biome", "tooling", "dependencies"],
      "always": true
    },
    {
      "id": "general-scaffolding",
      "file": "general/scaffolding.md",
      "category": "general",
      "tags": ["scaffolding", "init", "turborepo", "next", "vite", "astro", "elysia"],
      "always": true
    },
    {
      "id": "general-monorepo",
      "file": "general/monorepo.md",
      "category": "general",
      "tags": ["turborepo", "monorepo", "bun", "workspaces"],
      "always": false,
      "when": "monorepo"
    },
    {
      "id": "frontend-react-components",
      "file": "frontend/react-components.md",
      "category": "frontend",
      "tags": ["react", "components", "typescript", "atomic-design", "fsd"],
      "always": false,
      "when": "frontend"
    },
    {
      "id": "frontend-react-state",
      "file": "frontend/react-state.md",
      "category": "frontend",
      "tags": ["react", "state", "zustand", "tanstack-query", "forms"],
      "always": false,
      "when": "frontend"
    },
    {
      "id": "frontend-nextjs",
      "file": "frontend/nextjs.md",
      "category": "frontend",
      "tags": ["nextjs", "react", "rsc", "ssg", "isr"],
      "always": false,
      "when": "nextjs"
    },
    {
      "id": "backend-structure",
      "file": "backend/structure.md",
      "category": "backend",
      "tags": ["structure", "modules", "typescript", "elysia"],
      "always": false,
      "when": "backend"
    },
    {
      "id": "backend-elysia",
      "file": "backend/elysia.md",
      "category": "backend",
      "tags": ["elysia", "bun", "api", "routing"],
      "always": false,
      "when": "elysia"
    },
    {
      "id": "backend-database-drizzle",
      "file": "backend/database-drizzle.md",
      "category": "backend",
      "tags": ["drizzle", "postgresql", "database", "repository"],
      "always": false,
      "when": "drizzle"
    },
    {
      "id": "backend-api-design",
      "file": "backend/api-design.md",
      "category": "backend",
      "tags": ["api", "rest", "errors", "responses"],
      "always": false,
      "when": "backend"
    }
  ]
}
```

### `always` field

- `true` — rule is included in every project regardless of stack
- `false` — rule is included only when the `when` condition matches the project type

### `when` values

| Value | Included when |
|---|---|
| `frontend` | Project has a frontend |
| `backend` | Project has a backend |
| `nextjs` | Frontend framework is Next.js |
| `elysia` | Backend framework is Elysia |
| `drizzle` | Project uses Drizzle ORM |
| `monorepo` | Project is a monorepo |

---

## Contributing

1. Keep each rule file focused on a single topic
2. Always include YAML frontmatter with `id`, `category`, `tags`, and `description`
3. Update the manifest in `README.md` when adding or removing rule files
4. Write rules in English, imperative tone — "Use X", "Never do Y"
5. Include ✅ / ❌ code examples where applicable
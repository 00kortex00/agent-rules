---
id: design-pencil-dev
category: design
tags: [pencil-dev, design, figma, ui, components, tokens, auto-layout]
description: Rules for working with Pencil.dev — file structure, component extraction, auto layout, and AI prompt workflow
---

# Pencil.dev — Design Rules

Rules for AI agents generating or editing Pencil.dev designs.  
Always read these before writing any Pencil.dev prompt or modifying a design file.

---

## 1. File Structure

Every Pencil.dev file is divided into two root sections: `pages/` and `components/`.  
Each root section uses **vertical auto layout** (column, gap 80px).  
Groups inside each root use **horizontal auto layout** (row, gap 40px, align top).

```
pages/                          ← vertical auto layout, gap 80px
├── <FeatureA>/                 ← horizontal auto layout, gap 40px
│   ├── FeatureA — State1
│   ├── FeatureA — State2
│   └── FeatureA — State3
└── <FeatureB>/                 ← horizontal auto layout, gap 40px
    └── FeatureB — Overview

components/                     ← vertical auto layout, gap 80px
├── Shared/                     ← horizontal auto layout, gap 40px
│   └── Sidebar, Topbar, Button, Input, Badge, Modal, DataTable...
├── <FeatureA>/                 ← horizontal auto layout, gap 40px
│   └── FeatureA-specific components
└── <FeatureB>/                 ← horizontal auto layout, gap 40px
    └── FeatureB-specific components
```

### Frame naming convention

```
<Section> — <Variant or State>
```

Examples:
- `Users — List`
- `Users — Detail`
- `Settings — Danger Zone`
- `Dashboard — Overview (empty)`

Use ` — ` (space + em dash + space) as a separator. Never use `/` or `_`.

---

## 2. Components — when to extract, when to keep inline

**Extract to `components/`** only when an element appears in **2 or more** frames.  
If an element is unique to a single frame — describe it inline; do not create a component.

```
✅ Extract:  elements used in 2+ places (cards, rows, badges, modals)
❌ Keep inline: one-off panels, drop zones, unique overlays
```

**Placement rules:**

- Used in one section only → goes into that section's group in `components/`
- Used across multiple sections → goes into `Shared/`

**Before describing any new element**, check whether a matching component exists in `Shared/`.  
Shared elements (Sidebar, Topbar, Buttons, Inputs, Badge, Modal, DataTable, etc.) must  
always be **referenced**, never redrawn.

---

## 3. Auto Layout — principles

- Use auto layout wherever elements are arranged in a row or column.
- Use absolute positioning **only** for overlays: modals, tooltips, hover-state panels on top of cards.
- All spacing and gap values must come from the project spacing scale:

```
4  8  12  16  20  24  32  40  48  64
```

Never use arbitrary values like `10`, `15`, `22`, etc.

### Fixed dimensions

| Element       | Fixed size        |
|---------------|-------------------|
| Sidebar       | width 220px       |
| Topbar        | height 52px       |
| Modal         | width 480px       |
| Detail Panel  | width 280px       |

Everything else: `fill` (grow to parent) or `hug` (shrink to content).

---

## 4. Visual Hierarchy

- **One `btn-primary` per frame.** All secondary actions use `btn-default` or `btn-ghost`.
- **Destructive actions** use `btn-danger` only, and always require a Modal confirmation dialog.
- **Statuses and labels** are shown only via `Badge` components — never raw colored text.
- Use the surface token stack for depth: `surface → elevated → overlay`.  
  Do not use shadows — depth is achieved through background tokens only.

---

## 5. Design Tokens — hard constraints

Never introduce values outside the project's design system.

| Token type  | Rule |
|-------------|------|
| Colors      | Only tokens from the color system. No custom hex/rgb values. |
| Typography  | Only sizes and weights from the type scale. |
| Spacing     | Only values from the spacing scale (see §3). |
| Shadows     | **None.** Use bg-tokens for depth instead. |
| Gradients   | **None.** All backgrounds are solid. |

---

## 6. What Not To Do

```
❌ Introduce new colors outside the token system
❌ Introduce new font sizes outside the type scale
❌ Add box shadows
❌ Use gradients on backgrounds
❌ Redraw Sidebar / Topbar on each frame — reference Shared/ component
❌ Create a component for a single-use element
❌ Use absolute positioning for layout (only for overlays)
❌ Use arbitrary spacing values not in the scale
```

---

## 7. AI Prompt Workflow for Pencil.dev

When writing a prompt for Pencil.dev, structure it in this order:

### Step 1 — Name the frame

```
Frame name: <Section> — <State>
Example: Frame name: Orders — List (empty state)
```

### Step 2 — Describe layout

Specify the top-level auto layout direction, then describe inner groups the same way.

```
Top level: vertical auto layout, gap 24px, padding 24px
Row: horizontal auto layout, gap 16px, align center
```

### Step 3 — Reference existing components

Always call out which shared components are used before describing new elements.

```
Use from Shared/:
- Sidebar (left, fixed 220px)
- Topbar (top, fixed 52px)
- DataTable (full width)
```

### Step 4 — Describe new or inline elements

Only after referencing shared components, describe what is unique to this frame.

```
Inline elements:
- Empty state illustration, centered, hug content
- Subtitle text: "No orders yet", text-secondary, body-md
- btn-primary "Create order", centered below subtitle
```

### Step 5 — Confirm component extraction

If any new element should be extracted to `components/`, say so explicitly:

```
Extract to components/Orders/: Order Row (used in list and detail views)
```

---

## 8. Context files

When working on a specific project, the design rule file is meant to be used  
together with a project-specific context file that provides:

- The actual token values (color palette, spacing scale, type scale)
- The full list of available Shared/ components with their props
- Any project-specific layout constraints

Name the context file `<project>-design-context.md` and keep it alongside this file  
in the project's `_designContext/` directory.

# Project Pulse Dashboard — Implementation Plan

## Overview

**Project Pulse** is a small, static, client-side dashboard that visualizes a portfolio of projects (status, health, progress, priority, owner, etc.) from a local JSON data file. It is intentionally dependency-free: plain HTML, CSS, and a single `fetch()` call to load `project-data.json`. The app will be runnable and debuggable from VS Code via a `launch.json` browser-debug configuration, backed by a simple local static server so `fetch()` works reliably (avoiding `file://` CORS issues).

Scope of this plan:

- `app/index.html` — semantic markup and data-binding JS (Coder)
- `app/styles.css` — visual system, layout, responsive rules (Designer)
- `app/project-data.json` — data schema + realistic sample data (Coder)
- `.vscode/launch.json` — VS Code debug/preview launch config (Coder)

The plan aligns with the repo's agent roles defined in `docs/agent-team.md` (Orchestrator, Planner, Designer, Coder) and preserves non-overlapping file ownership so Designer and Coder can execute in parallel where safe.

---

## File Assignments

| File | Owner | Action | Rationale |
|---|---|---|---|
| `app/index.html` | **Coder** | Create | Markup structure, data-loading script, DOM wiring. Structure follows the IA/contract defined by the Designer. |
| `app/styles.css` | **Designer** | Create | Visual system, typography, color tokens, status/health color coding, card layout, responsive rules. |
| `app/project-data.json` | **Coder** | Create | Deterministic schema and seed data. Field names must match the design contract so CSS class hooks line up. |
| `.vscode/launch.json` | **Coder** | Create | Local browser-based debug/preview session. Must be authored after the app's serving strategy is decided. |

Supporting (implicit) items — no separate file, tracked as plan artifacts:

- **Design contract** (one-pager the Designer produces first) — enumerates CSS class hooks, `data-*` attributes, status vocabulary, and card component structure. Consumed by both Coder (for HTML/JSON) and Designer (for CSS).

No modifications to existing files (`README.md`, `.vscode/tasks.json`, `docs/agent-team.md`, `.devcontainer/*`) are required.

---

## Designer Responsibilities

**Owns:** `app/styles.css` + the *design contract* handed to the Coder.

1. **Information architecture / layout**
   - Top-level regions: header (title "Project Pulse", last-updated timestamp, summary KPIs), filter/legend row (optional, static), main grid of project cards, footer.
   - Project card anatomy: title, owner, status badge, health indicator (dot or ring), progress bar with numeric %, priority tag, due date, short description, tag chips.
   - Responsive grid: 1 col mobile (<640px), 2 col tablet, 3–4 col desktop, using CSS Grid `auto-fit` + `minmax`.

2. **Visual system**
   - Define CSS custom properties (`:root`) for color tokens, spacing scale, border radius, shadows, font stack.
   - Support system-preference dark mode via `@media (prefers-color-scheme: dark)`.
   - Font stack: system UI fonts only (no external font requests → keeps the app fully offline-capable).

3. **Status & health color coding** (canonical vocabulary — becomes the *design contract*):
   - **Status**: `on-track` (green), `at-risk` (amber), `off-track` (red), `blocked` (purple), `completed` (blue/neutral), `not-started` (gray).
   - **Health**: `green` / `yellow` / `red` (independent from status; e.g., completed+green).
   - **Priority**: `p0` / `p1` / `p2` / `p3` with escalating emphasis.
   - Provide CSS class hooks: `.status--on-track`, `.status--at-risk`, `.health--green`, `.priority--p0`, etc.
   - Ensure WCAG AA contrast (≥ 4.5:1) for status text on badge backgrounds.

4. **Progress bar spec**
   - Semantic `<progress>` or a styled `div` with `role="progressbar"` + `aria-valuenow/min/max`.
   - Fill color derives from health class, not status.

5. **UX guidance handed to Coder for `index.html`**
   - Required landmark elements: `<header>`, `<main>`, `<section>`, `<footer>`.
   - Card should be an `<article>` with an `aria-labelledby` pointing to the card title.
   - Provide the *exact* class names, `data-*` attributes, and card DOM skeleton the Coder must render (see contract).
   - Include an empty-state block and an error-state block (hidden by default; toggled by Coder JS).

6. **Deliverables**
   - `app/styles.css` implementing the above.
   - A short "design contract" comment block at the top of `styles.css` (also mirrored to the Orchestrator in the handoff) that the Coder uses as the source of truth for markup + JSON keys.

---

## Coder Responsibilities

**Owns:** `app/index.html`, `app/project-data.json`, `.vscode/launch.json`.

1. **`app/project-data.json` — schema**
   ```
   {
     "generatedAt": "ISO-8601 string",
     "projects": [
       {
         "id": "string (kebab-case, unique)",
         "name": "string",
         "owner": "string",
         "status": "on-track | at-risk | off-track | blocked | completed | not-started",
         "health": "green | yellow | red",
         "priority": "p0 | p1 | p2 | p3",
         "progress": 0-100 (integer),
         "dueDate": "YYYY-MM-DD",
         "description": "string (<= 160 chars)",
         "tags": ["string", ...]
       }
     ]
   }
   ```
   - Provide 6–9 seed projects covering every status/health/priority value at least once (so Designer's CSS is visually exercised).
   - Values must match the vocabulary in the design contract exactly (case-sensitive).

2. **`app/index.html` — structure & wiring**
   - Semantic landmarks per design contract.
   - Inline `<script type="module">` (or plain `<script>`) that:
     1. `fetch('./project-data.json')` on `DOMContentLoaded`.
     2. Renders one card per project by cloning a `<template id="project-card-template">` block.
     3. Populates KPIs in header (total, on-track count, at-risk count, avg progress).
     4. On fetch failure, shows the Designer-provided error state and logs to console.
     5. On empty `projects[]`, shows the empty state.
   - No frameworks, no build step, no external CDNs.
   - Link `styles.css` via `<link rel="stylesheet" href="./styles.css">`.
   - Include `<meta charset>`, `<meta name="viewport">`, `<title>Project Pulse</title>`.

3. **`.vscode/launch.json` — local debug/preview**
   - Use built-in `chrome` (or `msedge`) debug type with `"request": "launch"`.
   - Serve via a lightweight static server so `fetch()` works (avoid `file://`). Two acceptable approaches — Coder picks one and documents it:
     - **Option A (recommended, zero-install):** `preLaunchTask` that runs `python3 -m http.server 5500 --directory app` (Python 3 ships in the devcontainer/macOS). Add matching entry to `.vscode/tasks.json` **only if needed** — otherwise, use `serverReadyAction` with an inline `runtimeExecutable` is not supported; prefer a task.
     - **Option B:** Recommend the "Live Server" extension and target `http://127.0.0.1:5500/app/index.html` (no preLaunchTask). Document the extension dependency in the launch config comment.
   - Config shape:
     ```
     {
       "version": "0.2.0",
       "configurations": [
         {
           "type": "chrome",
           "request": "launch",
           "name": "Launch Project Pulse",
           "url": "http://127.0.0.1:5500/index.html",
           "webRoot": "${workspaceFolder}/app",
           "preLaunchTask": "serve-project-pulse"   // only if Option A
         }
       ]
     }
     ```
   - If Option A is chosen, Coder must also add a `serve-project-pulse` task to `.vscode/tasks.json` **without removing** the existing `Open Copilot CLI exercise terminal` task. This is the *one* case where Coder touches an existing file — flag to Orchestrator.

4. **Deliverables**: three new files, plus (conditionally) a task addition, all validated locally.

---

## Dependencies

1. **Design contract → everything else.** The vocabulary (status/health/priority values), CSS class hooks, and card DOM skeleton must be fixed before Coder writes `index.html` or `project-data.json`, because JSON field values feed CSS classes (e.g., `class="status status--${project.status}"`).
2. **`project-data.json` schema → `index.html` render logic.** JS accesses specific keys; schema must be locked first (can be defined by Coder immediately after design contract).
3. **`styles.css` ↔ `index.html`** are coupled by class names and `data-*` attributes but not by execution order — as long as both sides honor the design contract, they can be written in parallel.
4. **Serving strategy decision → `.vscode/launch.json`.** Coder must decide Option A vs Option B before writing `launch.json`. This decision is independent of Designer work.
5. **Integration verification → all four files complete.** Orchestrator can only verify end-to-end after Designer + Coder both report done.

---

## Parallel Work Decisions

| Track | Files | Can run in parallel with | Why |
|---|---|---|---|
| **T0 — Design contract** | (spec artifact) | — must run first | Locks vocabulary that both HTML/JSON and CSS depend on. Short (~single deliverable). |
| **T1 — Designer: styles.css** | `app/styles.css` | T2, T3, T4 | No file overlap with Coder's files. Depends only on T0. |
| **T2 — Coder: project-data.json** | `app/project-data.json` | T1, T3, T4 | No overlap; schema derived from T0. |
| **T3 — Coder: index.html** | `app/index.html` | T1, T2, T4 | No file overlap. Uses contract from T0 and schema from T2 (schema is trivial and can be agreed up front, so T2 and T3 need not be strictly ordered). |
| **T4 — Coder: launch.json (+ optional tasks.json edit)** | `.vscode/launch.json`, possibly `.vscode/tasks.json` | T1, T2, T3 | No overlap with Designer. Independent of T0's vocabulary. |

**Must be sequential:**
- T0 **before** T1, T2, T3 (vocabulary lock).
- Orchestrator integration check **after** T1–T4.

**Must not run in parallel:**
- Any two tasks that touch `.vscode/tasks.json`. Only T4 may touch it, and only if Option A is chosen. Designer never touches `.vscode/*`.

**Safely parallel:**
- T1 (Designer CSS) with T2+T3+T4 (Coder) — disjoint file scope.
- Within Coder's own bucket, T2, T3, T4 can also be authored in parallel because they are three separate files with no cross-writes; only the design contract couples T2 and T3 semantically (not textually).

---

## Phases / Execution Order

**Phase 1 — Contract lock (sequential, blocking)**
- Owner: Designer (produces contract) with Coder sign-off on the JSON schema.
- Output: design contract doc (vocabulary, class hooks, card DOM skeleton, JSON schema).
- Exit criteria: Orchestrator approves; both agents acknowledge.

**Phase 2 — Parallel build**
- Track A (Designer): write `app/styles.css`.
- Track B (Coder): write `app/project-data.json`, `app/index.html`, `.vscode/launch.json` (+ optional `.vscode/tasks.json` task entry if Option A).
- Exit criteria: each track self-validates (see Validation).

**Phase 3 — Integration verification (sequential)**
- Orchestrator (or learner) launches the debug configuration and walks the validation checklist.
- Any defect → routed back to the owning agent; other agent is idle.

**Phase 4 — Report & handoff**
- Orchestrator reports final outcome. Learner performs any git operations via Copilot CLI (agents do not stage/commit/push).

---

## Edge Cases to Handle

1. **`fetch()` fails or returns non-200** (e.g., served from `file://`, wrong path). → Show error state, log to console. This is why the launch config uses a real HTTP server.
2. **Empty `projects` array.** → Show empty state block (Designer provides styling; Coder toggles visibility).
3. **Unknown status/health/priority value** in JSON (typo). → Fall back to a neutral badge (`.status--unknown`) and log a `console.warn`. Designer must define `--unknown` fallback styles.
4. **Progress out of range** (negative or >100). → Clamp to [0, 100] in JS before rendering.
5. **Missing optional fields** (`tags`, `description`). → Render conditionally; no `undefined` in DOM.
6. **Past `dueDate`.** → Optional visual affordance (Designer's call — e.g., red date text). Not required for MVP.
7. **Dark mode toggle** via `prefers-color-scheme`. Verify both themes render.
8. **Long project names / descriptions.** → CSS must handle overflow (ellipsis or wrap) without breaking card grid.
9. **Port 5500 already in use** in the devcontainer. → Coder documents an alternate port in `launch.json` comments; task uses the same port.
10. **Live Server extension not installed** (Option B). → `launch.json` should include a top-level `// comment` (JSONC) explaining the prerequisite, or Coder chooses Option A to eliminate the dependency.
11. **Accessibility regressions.** → Every interactive/semantic element needs proper ARIA; progress bars need `aria-valuenow`; color must not be the *only* signal (pair with text labels).
12. **JSON authoring errors** (trailing commas, wrong quotes). → Coder validates with `python3 -m json.tool app/project-data.json` before handoff.

---

## Validation Expectations

**Static / per-file checks (each agent runs before handoff):**
- `app/project-data.json` → passes `python3 -m json.tool` (or `jq .`).
- `app/index.html` → opens without console errors; no 404s in Network tab for `styles.css` / `project-data.json`.
- `app/styles.css` → no unknown property warnings in DevTools; all documented class hooks exist.
- `.vscode/launch.json` → valid JSONC; VS Code shows the config in the Run & Debug dropdown.

**Integration checklist (Orchestrator or learner runs):**
1. Open the repo in VS Code → Run & Debug → select **"Launch Project Pulse"** → session starts, browser opens.
2. Dashboard renders header, KPI summary, and one card per project in `project-data.json`.
3. Status badges, health indicators, priority tags, and progress bars all display with the correct color coding for every vocabulary value (verified because seed data covers all values).
4. Responsive: resize window; grid reflows to 1/2/3+ columns at expected breakpoints.
5. Dark mode: toggle OS preference; theme updates.
6. DevTools Console is clean (no errors, no unresolved warnings other than intentional `console.warn` for unknown values).
7. DevTools Network: `styles.css` (200), `project-data.json` (200), no external requests.
8. Break-glass tests: temporarily rename `project-data.json` → error state visible; restore → cards return. Temporarily empty `projects: []` → empty state visible.
9. Breakpoint test: set a breakpoint in the `<script>` inside VS Code; reload; breakpoint hits → confirms `launch.json` source-mapping/`webRoot` is correct.
10. Accessibility spot-check: Tab order reaches interactive elements; progress bars announce values; contrast passes for badges.

**Definition of Done:**
- All four files exist and pass their static checks.
- Full integration checklist passes.
- Orchestrator posts a summary; learner (not agents) performs git operations.

---

## Open Questions

1. **Serving strategy** — Option A (Python http.server + preLaunchTask, requires editing `.vscode/tasks.json`) or Option B (Live Server extension, adds an implicit dependency)? Recommendation: **Option A** for reproducibility in the devcontainer; needs Orchestrator confirmation that appending to `tasks.json` is acceptable.
2. **KPI summary in header** — required or optional? Plan currently includes it as a lightweight nice-to-have; can be dropped without impact if the Designer wants a minimalist header.
3. **Filters/search** — out of scope for MVP per the requirements (static dashboard). Confirm we should *not* build interactive filtering.
4. **Data freshness** — is `generatedAt` displayed as-is, or humanized ("2 hours ago")? Humanization adds a small JS helper; default is as-is ISO string.
5. **Number of seed projects** — plan proposes 6–9 to cover all enum values. Confirm acceptable.
6. **Browser target** — `chrome` vs `msedge` in `launch.json`. Chrome is the safer default across macOS/Codespaces; confirm.
7. **Dark mode** — auto (via `prefers-color-scheme`) only, or also a manual toggle button? Plan assumes auto-only.

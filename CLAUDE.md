# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file demo/training Kanban board for UOB's internal IT PMO (`index.html`). It is a deliberately constrained vanilla-JS build — not a scaffold for a larger app. Preserve the constraints below when making changes; do not introduce a build step, framework, or backend as a "fix."

## Running / testing

There is no build, install, or test command. Open `index.html` directly in a browser (double-click, or `open index.html` on macOS). There is no dev server, linter, or package.json — none should be added.

To sanity-check changes without a browser, verify the inline `<script>` is syntactically valid and that every `getElementById("...")` call in JS resolves to an `id="..."` in the HTML (static ids only — a few, like `error-<field>`, are built by string concatenation in `setFieldError`/`resetForm` and won't show up in a plain id diff).

## Hard constraints (do not violate)

- **Vanilla HTML/CSS/JS only.** No React, Vue, jQuery, Tailwind, bundler, or npm dependencies of any kind.
- **Single file.** All markup, the `<style>` block, and the `<script>` block live in `index.html`. Do not split into separate `.css`/`.js` files.
- **No persistence.** Board state is an in-memory JS object only — no `localStorage`, `sessionStorage`, `IndexedDB`, or cookies. A page refresh is *meant* to reset the board to seed data; the UI's "demo note" banner communicating this must stay accurate if state handling changes.
- **No external resources.** No CDN scripts/styles, no Google Fonts, no image files. Icons are Unicode glyphs or inline SVG; fonts are the system stack (`--font-stack` custom property).
- **FormSubmit is the only network call**, and it's fire-and-forget: a failed request must never affect board state, only surface a warning toast. Never send the endpoint anything beyond the task fields it's given.
- **No UOB trademarks/logo.** The header is a plain "UOB IT PMO" text wordmark on a neutral corporate-blue palette — not a recreation of any real UOB branding or system.

## Architecture

Everything hangs off one module-level `state` object (`tasks`, `filters`, `pendingDeleteId`, `idCounter`) defined near the top of the `<script>` block. There is no other source of truth: **all rendering flows one-way from `state` → `renderBoard()` → DOM**, and DOM event handlers mutate `state` and then call `renderBoard()` again rather than patching individual card elements. When adding a feature, extend `state` and re-render rather than reading values back out of the DOM.

Key functions and their roles:
- `renderBoard()` — rebuilds all four columns and cards from `getFilteredTasks()`, then calls `renderSummary()` and re-wires `attachBoardEventListeners()`. Because it does a full `innerHTML` rewrite of `#board`, event listeners on cards/columns must be re-attached every call (they're delegated per-render, not once at init).
- `renderCard(task)` — pure string-template function; any user-supplied field going into HTML must go through `escapeHtml()` first (this includes seed data fields too, for consistency).
- `moveTask` / `deleteTask` / `addTask` — the only functions that mutate `state.tasks`. Both drag-and-drop (`dragstart`/`dragover`/`drop` handlers) and the keyboard-accessible "Move ▸" `<select>` on each card call the same `moveTask()`, so any new move pathway should too.
- Inline delete confirmation (`request-delete` → `confirm-delete`/`cancel-delete`) is tracked via `state.pendingDeleteId`, not a native `confirm()` dialog — a re-render shows the Yes/No toggle only on the matching card.
- `notifyNewTask(task)` — the sole network call, wrapped in try/catch at the call site in `handleAddTaskSubmit`; a rejection must only produce a toast, never roll back the optimistically-added card.
- `showToast()` — appends to the `aria-live="polite"` toast region and self-removes after a timeout; there's no toast queue/limit logic.

Config: `FORMSUBMIT_ENDPOINT` is a single constant near the top of the script — this is the one place to change the notification recipient. FormSubmit requires a one-time activation (a confirmation email sent to that address on first submission) before notifications actually deliver; this is noted in an HTML comment directly above the form.

## Data model

```js
{
  id: "UOB-ITPM-0001",       // generated via generateTaskId(), zero-padded counter
  title, description,
  project,                    // one of 6 fixed workstreams (see <select> options)
  category,                   // one of 6 fixed categories
  assignee, priority,         // Critical | High | Medium | Low
  dueDate,                    // ISO yyyy-mm-dd
  status,                     // Backlog | In Progress | Blocked | Done — must match STATUSES exactly
}
```

`STATUSES` (the four fixed columns) is the single array driving column order everywhere — column rendering, the "Move ▸" `<select>` options, and the summary strip counts all iterate over it. Priority-to-color mapping (border, pill) is done via CSS classes derived from `priority.toLowerCase()` (`.priority-critical`, `.pill-critical`, etc.), so priority values must stay capitalized exactly as `Critical | High | Medium | Low`.

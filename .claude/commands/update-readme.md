---
description: Create README.md if it doesn't exist, or update it to match the current code
argument-hint: [optional: what to add or change in the README]
---

Create or update `README.md` at the root of this repository.

Extra instructions from the user (may be empty): $ARGUMENTS

Follow these steps:

1. **Read the project first.** Read `CLAUDE.md` and `index.html` (markup, `<style>`, and `<script>`), and list the repo files. Base the README only on what the code actually does — don't invent features, commands, or dependencies.

2. **Create or edit:**
   - **No `README.md`:** create one with the sections below.
   - **`README.md` exists:** read it, keep the user's own wording and any sections they added, and update only what is outdated, missing, or contradicted by the code. Don't rewrite it from scratch.

3. **Sections** (adapt to what's relevant):
   - **Title and one-line summary** — a single-file demo/training Kanban board for an internal IT PMO.
   - **Features** — the four columns (from `STATUSES`), add task form, drag-and-drop plus the keyboard "Move ▸" select, filters, summary strip, inline delete confirmation, toast notifications, FormSubmit email notification on new tasks.
   - **Getting started** — open `index.html` in a browser. No install, build, or server.
   - **Configuration** — `FORMSUBMIT_ENDPOINT` in the script, and the one-time FormSubmit activation email.
   - **Data model** — the task fields, priority values (`Critical | High | Medium | Low`), and status values.
   - **Limitations by design** — no persistence (refresh resets to seed data), single file, vanilla HTML/CSS/JS only, no external resources.
   - **Project structure** — the files in the repo, including `.claude/commands/` if present.
   - **Disclaimer** — demo/training only; not affiliated with or a recreation of any real UOB system or branding.

4. **Respect the project rules in `CLAUDE.md`.** The README must not suggest adding a framework, build step, backend, or persistence, and must not include real email addresses or secrets beyond what's already in the code.

5. **Show the result.** Summarize what you created or changed. Ask the user before committing; if they agree, commit with a clear message (e.g. "Add README" or "Update README"). Push only if the user asks, or suggest `/push-to-github`.

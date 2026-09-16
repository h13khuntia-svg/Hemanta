---
description: Security-scan, push this repo to a GitHub URL, stand up GitHub Pages via a workflow, and write the README + About section
argument-hint: <github-repo-url> [target-branch]
---

## Context

- Repo URL to deploy to: `$1`
- Target branch (optional, defaults to `main`): `$2`
- Current local branch: !`git branch --show-current`
- Local remotes: !`git remote -v`
- Working tree status: !`git status --short`
- Files tracked in this repo: !`git ls-files`

## Goal

Take this repository (a single-file vanilla-JS Kanban board, see @CLAUDE.md) and publish
it end-to-end to the GitHub repo URL the user supplied as `$1`: push the code, stand up
GitHub Pages through a GitHub Actions workflow, write a README, and fill in the repo's
About section (description + homepage link) — but only *after* a security scan confirms
nothing sensitive is going out to the public internet.

If `$1` is missing or is not a plausible `https://github.com/<owner>/<repo>` (or
`git@github.com:<owner>/<repo>.git`) URL, stop and ask the user for it before doing
anything else. Do not guess a repository.

Work through the steps below **in order**. Do not skip the security scan and do not push
if it fails. Give the user a short status update after each numbered step, not just at
the end.

## Step 1 — Security scan (blocking)

This must pass before anything is pushed anywhere.

1. Enumerate every tracked file (`git ls-files`) plus anything currently staged/untracked
   that would be swept up by a commit — the demo is a single `index.html`, but check the
   whole tree, not just that file, in case other files have been added since.
2. Read each file and scan for secrets and sensitive data:
   - Run `mcp__github__run_secret_scanning` against the raw contents of each file (pass
     file contents, not paths).
   - Additionally grep for patterns the scanner might miss in a demo context: private key
     blocks (`BEGIN...PRIVATE KEY`), AWS-style keys (`AKIA...`), bearer tokens, `password`/
     `passwd`/`secret`/`api_key`/`apikey` assigned to a literal value, `.env` style
     `KEY=value` lines, connection strings (`mongodb://`, `postgres://`, etc. with embedded
     creds), and real-looking personal data beyond what the app intentionally displays as
     seed/demo data.
   - Check `git log -p` for the same patterns — a secret removed in a later commit is still
     public once pushed, since it lives in history.
3. Specifically sanity-check `FORMSUBMIT_ENDPOINT` in `index.html`. Per @CLAUDE.md this is
   the *one* deliberate network call and is meant to be a visible client-side constant —
   that's expected and not a finding on its own. Flag it only if it looks like it points at
   an address the user wouldn't want public (e.g. a personal inbox they didn't expect to be
   in this file) — mention it to the user as an FYI either way, since it becomes
   world-readable the moment this pushes.
4. Confirm there's no `.env`, credentials file, private key, IDE config with tokens, or
   `node_modules`/build output accidentally tracked (none should exist per the project's
   constraints, but verify rather than assume).
5. Report findings:
   - **Nothing found** → say so briefly and proceed to Step 2.
   - **Real secret found** → STOP. Do not push. Tell the user exactly what was found and
     where (file + line), and that it must be removed/rotated before this command can
     continue. Do not attempt to auto-redact and push anyway.
   - **Ambiguous/low-severity** (e.g. an email address, a name) → surface it, use judgement,
     and only proceed if it's consistent with this being a public demo (per @CLAUDE.md this
     is a training/demo Kanban board with fictional seed data — placeholder names/emails in
     seed data are fine; anything that looks like a real person's private contact info tied
     to the *submitter* rather than the demo is not).

## Step 2 — Push the code

1. Parse `owner` and `repo` out of `$1`.
2. If a `origin` remote already exists and points somewhere else, don't silently overwrite
   it — tell the user what's currently configured and confirm before changing it. If it
   already points at the same URL, reuse it.
3. `git remote add origin $1` (or `git remote set-url origin $1` if it already exists and
   the user confirmed overwriting it).
4. Push the current branch to the target branch (`$2`, default `main`) on that remote:
   `git push -u origin HEAD:<target-branch>`.
   - If push fails because the remote has unrelated history (e.g. the user created the repo
     with an auto-generated README/license), fetch it, and merge rather than force-push:
     never `push --force` here without the user explicitly asking for it.
   - If push fails on auth, tell the user plainly what failed rather than trying to work
     around credentials.

## Step 3 — GitHub Pages via GitHub Actions

This is a static single-file site, so Pages needs nothing built — just published.

1. Add `.github/workflows/deploy-pages.yml` (create or update) with a minimal workflow that:
   - Triggers on `push` to the target branch (and `workflow_dispatch` for manual re-runs).
   - Has `permissions: contents: read, pages: write, id-token: write`.
   - Uses a single `concurrency` group (`pages`, `cancel-in-progress: false`) so overlapping
     deploys queue instead of racing.
   - Job 1 (`build`): `actions/checkout@v4`, then `actions/configure-pages@v5`, then
     `actions/upload-pages-artifact@v3` with `path: .` (the whole repo root, since
     `index.html` lives at the top level — do not introduce a build step per @CLAUDE.md).
   - Job 2 (`deploy`): `needs: build`, `environment: github-pages`, uses
     `actions/deploy-pages@v4` and exposes the resulting `page_url` output.
2. Commit and push this workflow file to the target branch (same remote/branch as Step 2).
3. GitHub Pages must have its **source** set to "GitHub Actions" (not "Deploy from a
   branch") for this workflow to succeed. There is no API tool available in this session to
   flip that setting, so:
   - Try `mcp__github__actions_list` (`list_workflow_runs`) after pushing to see if the
     workflow ran and check its conclusion via `mcp__github__actions_get`.
   - If it failed specifically because Pages isn't configured for Actions yet (a common
     first-run error), tell the user to do this one manual step: repo **Settings → Pages →
     Build and deployment → Source → GitHub Actions**, then re-run the workflow (you can
     trigger a re-run via `mcp__github__actions_run_trigger` if available, or ask them to
     push again / re-run it from the Actions tab).
4. Once the workflow succeeds, the site URL is `https://<owner>.github.io/<repo>/`
   (project page) — confirm this matches the workflow's `page_url` output if you were able
   to read it. Note this URL; it's needed for Steps 4 and 5.

## Step 4 — README

Create or update `README.md` at the repo root (this is repository metadata, not part of the
app's single-file source, so it doesn't violate the single-file constraint in @CLAUDE.md).
Read the existing `README.md` first if one exists (don't clobber content blindly) and merge
in what's below rather than duplicating sections:

- Project title and one-paragraph description, pulled from @CLAUDE.md ("single-file demo/
  training Kanban board for UOB's internal IT PMO").
- A **live demo** link to the GitHub Pages URL from Step 3, once known.
- A short "what this is / what this isn't" section: vanilla HTML/CSS/JS, no build step, no
  backend, no persistence (state resets on refresh) — pull this straight from @CLAUDE.md so
  it can't drift out of sync with the actual constraints.
- How to run it locally: just open `index.html` in a browser.
- A note that the FormSubmit integration needs one-time activation (confirmation email) and
  is fire-and-forget — it never affects board state.
- Skip build/install/test instructions entirely — there are none, and don't invent any.

## Step 5 — About section (description + homepage link)

GitHub's repo "About" panel (description, website link, topics) has no dedicated API tool
available in this session. Handle it in this order:

1. Check whether a suitable tool exists now (tool availability can change) — search for one
   that updates repository metadata/description/homepage before assuming it's unavailable.
2. If no such tool is available, don't skip this step silently: give the user the exact
   values to paste in themselves, since you can't do it for them:
   - **Description**: one line, e.g. "Single-file demo Kanban board for UOB IT PMO training
     (vanilla HTML/CSS/JS, no backend)."
   - **Website**: the Pages URL from Step 3.
   - **Topics** (optional suggestion): `kanban`, `vanilla-js`, `demo`, `github-pages`.
   - Tell them exactly where: repo main page → gear/⚙️ icon next to "About" in the right
     sidebar → fill in Description and Website → Save.

## Final report

End with a concise status list covering all five steps (done / skipped / blocked and why),
the live Pages URL, and anything the user still needs to do manually (Pages source setting,
About section paste-in) if you couldn't complete it via available tools.

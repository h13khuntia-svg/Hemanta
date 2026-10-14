---
description: Security-scan the project, then push it to GitHub, update the README and repo "About", and deploy GitHub Pages via a GitHub Action
argument-hint: <owner/repo or repo URL> [branch=main] [visibility=public|private] [extra notes]
---

# /publish — scan, push, document, and deploy to GitHub Pages

Repo details from the user: **$ARGUMENTS**

You are publishing this project (a single-file vanilla-JS demo, `index.html` — see `CLAUDE.md`) to GitHub. Work through the phases below **in order**. The security scan is a hard gate: nothing leaves this machine until it passes.

## Phase 0 — Resolve the target

1. Parse `$ARGUMENTS` for:
   - **Repo**: `owner/repo`, `https://github.com/owner/repo(.git)`, or `git@github.com:owner/repo.git`. Normalise to `owner/repo`.
   - **Branch**: default `main`.
   - **Visibility** (only used if the repo has to be created): default `public` — GitHub Pages on a free plan needs a public repo.
2. If no repo was given, or it's ambiguous, ask the user before going further. Do not guess an owner.
3. Check tooling: `git --version`, `gh auth status`. If `gh` isn't installed or authenticated, use the GitHub MCP tools (`mcp__github__*`) if available; otherwise stop and tell the user to run `gh auth login`.
4. Check whether the repo exists (`gh repo view owner/repo --json name,visibility,defaultBranchRef,description,homepageUrl,repositoryTopics`). If it doesn't, tell the user it will be created with the chosen visibility and confirm before creating it in Phase 2.

## Phase 1 — Security scan (blocking gate)

Scan **everything that would be pushed**: the working tree *and* the full git history (`git log -p --all`), since history goes up too. Also scan any file you generate in later phases before it is committed.

Run the strongest tool available, falling back in this order:
- `gitleaks detect --source . --log-opts="--all" --redact -v` (and `gitleaks detect --no-git` for untracked files), or
- `trufflehog git file://. --only-verified=false`, or
- the manual checks below (always run these too — they cover project-specific risks the tools miss).

Manual checks (use Grep over tracked + untracked, non-ignored files, and `git log -p --all` for history):

| Check | What to look for |
|---|---|
| Secrets / tokens | `ghp_`, `gho_`, `github_pat_`, `AKIA[0-9A-Z]{16}`, `xox[baprs]-`, `sk-`, `sk-ant-`, `AIza`, `-----BEGIN .*PRIVATE KEY-----`, `password\s*[:=]`, `secret\s*[:=]`, `token\s*[:=]`, `api[_-]?key`, bearer tokens, connection strings with credentials |
| Sensitive files | `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.sqlite`, `*.db`, credentials/config dumps, `.DS_Store`, editor/OS junk, `.claude/settings.local.json` |
| Personal data | Real email addresses, phone numbers, staff names, NRIC/ID numbers. **In particular check `FORMSUBMIT_ENDPOINT` in `index.html`**: a placeholder like `YOUR_EMAIL@example.com` is fine; a real address will be public on GitHub and GitHub Pages — flag it and ask the user whether to publish it, or swap it for a placeholder / FormSubmit's hashed alias. |
| Internal info | Internal hostnames, intranet URLs, private IPs (`10.`, `172.16–31.`, `192.168.`), real project codes or confidential client data in seed tasks |
| Project constraints (`CLAUDE.md`) | No external `<script src>`/`<link href>` to CDNs or Google Fonts; no `localStorage`/`sessionStorage`/`indexedDB`/`document.cookie`; the only `fetch` is to FormSubmit; no UOB logo/trademark assets |
| Unsafe code | `eval(`, `new Function(`, `innerHTML` with user input that bypasses `escapeHtml()`, `document.write` |
| `.gitignore` | Exists and excludes `.env*`, keys, `.DS_Store`, `node_modules/`, `.claude/settings.local.json`. Create/extend it if missing. |

Report the results as a short table: check → pass / finding (file:line, secret values **redacted**).

- **Any secret/credential found** → STOP. Do not push. Explain where it is, and if it's in git history, explain that it must be removed from history *and rotated* (it should be considered compromised). Only continue once the user has fixed it and a re-scan is clean.
- **Personal data / internal info / constraint violations** → show them and ask the user to fix, or explicitly accept, before continuing.
- **Clean** → say so and continue.

## Phase 2 — Push the code

1. Ensure there is a local repo and the working tree is clean (commit pending changes only after confirming with the user what's included — never `git add` files the scan flagged).
2. Add or update the remote: `git remote add origin https://github.com/owner/repo.git` (or `git remote set-url origin ...` if `origin` points elsewhere — show the old URL and confirm before changing it).
3. If the repo doesn't exist, create it after confirmation: `gh repo create owner/repo --<visibility> --source . --remote origin`.
4. Push: `git push -u origin <branch>`. On network failure retry up to 4 times with backoff (2s, 4s, 8s, 16s).
5. **Never force-push** unless the user explicitly asks. If the remote has diverging history, stop and explain the options (merge, rebase, or push to a new branch).

## Phase 3 — Create / edit the README

Read `index.html` and `CLAUDE.md` so the README reflects what the app actually does. If `README.md` exists, update it in place — keep any user-written sections, only refresh outdated facts.

Include:
- Title + one-line summary and a **Live demo** link: `https://<owner>.github.io/<repo>/` (lower-case owner).
- Features (Kanban columns from `STATUSES`, drag-and-drop + keyboard "Move ▸", filters, summary strip, inline delete confirm, toasts, FormSubmit notification).
- How to run: open `index.html` in a browser — no build, no install.
- Notes / limitations: in-memory only (refresh resets to seed data), FormSubmit one-time activation, where to change `FORMSUBMIT_ENDPOINT`.
- Tech constraints (vanilla HTML/CSS/JS, single file, no external resources).
- Deployment: GitHub Pages via the Action in `.github/workflows/pages.yml`.
- Disclaimer: demo/training project, not an official UOB product or system; no UOB trademarks used.

Do not add badges or images that load from external services beyond the standard GitHub Actions status badge. Re-run the Phase 1 checks on the README, then commit (`docs: add/update README`).

## Phase 4 — Create / edit the GitHub repo "About"

Show the user the proposed values, then apply them with `gh repo edit owner/repo`:
- `--description` — one line, ≤ 350 chars (e.g. "Single-file vanilla-JS Kanban board demo for an IT PMO — drag-and-drop, filters, no build step.").
- `--homepage https://<owner>.github.io/<repo>/`
- `--add-topic` — e.g. `kanban`, `vanilla-javascript`, `html`, `css`, `github-pages`, `demo`, `project-management`. Remove stale topics only if the user agrees.

If the About already has a description, show the current vs proposed and only overwrite after confirmation.

## Phase 5 — Create / update GitHub Pages via a GitHub Action

1. Create or update `.github/workflows/pages.yml`. Publish **only the site files** (not `CLAUDE.md`, `.claude/`, etc.) by staging them into a `_site` folder:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]   # replace with the target branch
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - name: Stage site files
        run: |
          mkdir -p _site
          cp index.html _site/
          touch _site/.nojekyll
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site
      - id: deployment
        uses: actions/deploy-pages@v4
```

   If the workflow already exists, edit it rather than replacing it wholesale; keep permissions minimal (no `contents: write`, no secrets).
2. Enable Pages with the Actions source (idempotent):
   - Not yet enabled: `gh api -X POST repos/owner/repo/pages -f build_type=workflow`
   - Already enabled with branch source: `gh api -X PUT repos/owner/repo/pages -f build_type=workflow`
   - If this fails (e.g. private repo on a free plan), explain why and what the user needs to change.
3. Re-run the Phase 1 checks on the workflow file, commit (`ci: deploy to GitHub Pages`), and push.
4. Watch the run: `gh run list --workflow pages.yml --limit 1`, then `gh run watch <id> --exit-status`. On failure, read the logs (`gh run view <id> --log-failed`), fix, and push again.
5. Confirm the site responds: `curl -sI https://<owner>.github.io/<repo>/` (first deploy can take a minute or two).

## Phase 6 — Summary

Finish with a short report:
- Security scan result (and anything the user chose to accept).
- Repo URL, branch, and commit(s) pushed.
- README: created or updated.
- About: description, homepage, topics.
- Pages: workflow status and live URL.
- Any follow-ups (e.g. activate FormSubmit, rotate a secret, make repo public for Pages).

## Rules

- The security gate in Phase 1 is never skipped, even if the user says "just push".
- Confirm before anything hard to reverse or public: creating a repo, changing visibility, overwriting the About, changing `origin`, rewriting history.
- Never force-push, never commit files the scan flagged, never print secret values in full.
- Respect `CLAUDE.md`: don't add a build step, package.json, frameworks, or external resources to the app while doing this.

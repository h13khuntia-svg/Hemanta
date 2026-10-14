---
description: Security-scan, then push this project to GitHub with README, repo About, and a GitHub Pages site deployed by GitHub Actions
argument-hint: [GitHub repo URL or owner/repo] [branch]
---

Publish this project to GitHub. The user provides the repository details.

Repo details from the user (may be empty): $ARGUMENTS

Read `CLAUDE.md` first and respect its hard constraints throughout: no build step, framework, npm, `package.json`, persistence, external resources, or extra CSS/JS files. Never force-push, and never open a pull request unless the user asks.

## 1. Get the repository details

If any of these are missing from `$ARGUMENTS`, ask the user and wait for the answer. Never guess a repository.
- **Repository:** `https://github.com/owner/repo` or `owner/repo` (normalize to `https://github.com/owner/repo.git`)
- **Branch** to push to and deploy from (default: the repo's default branch, usually `main`)
- **Custom domain** for GitHub Pages (default: none, so the site is at `https://<owner>.github.io/<repo>/`)

Work out how to reach the GitHub API, in this order: the `gh` CLI if it is installed and `gh auth status` succeeds, then the GitHub MCP tools (`mcp__github__*`) if available, otherwise none. If none, the steps that need the API (6 and 7) become manual instructions for the user.

## 2. Create or edit the README

Read `index.html` (markup, `<style>`, `<script>`) and list the repo files. Describe only what the code really does.
- **No `README.md`:** create one.
- **`README.md` exists:** keep the user's wording and any sections they added; update only what is outdated, missing, or wrong.

Sections (adapt as relevant):
- Title and one-line summary
- Live demo: the GitHub Pages URL
- Features: the columns from `STATUSES`, add-task form, drag-and-drop plus the keyboard "Move ▸" select, filters, summary strip, inline delete confirmation, toasts, FormSubmit email on new tasks
- Getting started: open `index.html` in a browser; no install, build, or server
- Configuration: `FORMSUBMIT_ENDPOINT` and the one-time FormSubmit activation email
- Data model: task fields, priorities `Critical | High | Medium | Low`, statuses
- Limitations by design: no persistence (refresh resets to seed data), single file, vanilla HTML/CSS/JS, no external resources
- Project structure
- Disclaimer: demo/training only, not affiliated with or a recreation of any real UOB system or branding

Don't suggest adding a framework, backend, build step, or persistence. Don't include secrets or real email addresses.

## 3. Create or update the GitHub Pages workflow

Read everything in `.github/workflows/`. Edit an existing Pages workflow (one using `actions/deploy-pages`) in place, changing only what's needed; otherwise create `.github/workflows/pages.yml`:
- **Triggers:** `push` to the chosen branch, and `workflow_dispatch`.
- **Permissions:** `contents: read`, `pages: write`, `id-token: write`.
- **Concurrency:** group `pages`, `cancel-in-progress: false`.
- **`validate` job** (`ubuntu-latest`, only preinstalled tools, checks written inline in the workflow, no extra files), after `actions/checkout@v4`:
  - extract the inline `<script>` from `index.html` and run `node --check` on it
  - fail if any static `getElementById("...")` has no matching `id="..."` (ids built by string concatenation, like `error-<field>`, are dynamic; only check string literals)
  - fail on `localStorage`, `sessionStorage`, `indexedDB`, `document.cookie`, or any `<script src>`/`<link href>` to `http(s)://`
- **`build` job** (`needs: validate`): `actions/checkout@v4` → `actions/configure-pages@v5` → copy only the site files into `_site/` (`index.html`, plus `CNAME` and `.nojekyll` if present; never `.git`, `.claude/`, `.github/`, `CLAUDE.md`, or `README.md`) → `actions/upload-pages-artifact@v3` with `path: _site`.
- **`deploy` job** (`needs: build`): `environment: { name: github-pages, url: ${{ steps.deployment.outputs.page_url }} }`, `actions/deploy-pages@v4` with `id: deployment`.

If a custom domain was given, create or update `CNAME` with just the domain.

Check the YAML parses (e.g. `python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" .github/workflows/pages.yml` if PyYAML is available). Run the validate checks locally against the current `index.html`. If they fail, report why; don't loosen the checks.

## 4. Security scan (required before uploading anything)

This is read-only: change nothing during the scan. Scan everything that will leave this machine: every file `git add -A` would include, the unpushed commit history (`git log -p @{u}..HEAD`, or full history if there's no upstream), and the About text planned for step 6.

- **Secrets:** API keys, tokens, passwords, private keys (`-----BEGIN ... PRIVATE KEY-----`), `ghp_`/`gho_`/`github_pat_`, `AKIA`, `sk-`, `xox[bp]-`, connection strings, `.env`, `*.pem`, `*.key`, `id_rsa`.
- **Personal data:** email addresses, phone numbers, internal hostnames or IPs. The address in `FORMSUBMIT_ENDPOINT` is expected; report it as **info** because it becomes public on the Pages site.
- **`index.html`:**
  - every user-supplied or seed field going into `innerHTML` or template strings passes through `escapeHtml()`
  - no `eval`, `new Function`, `document.write`, or string `setTimeout`/`setInterval`
  - the only network call is the FormSubmit `fetch`, and it sends only the task fields (no other `fetch`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `<form action>`, or beacons)
  - no external resources, no browser storage
  - every `target="_blank"` has `rel="noopener noreferrer"`
- **Workflows:**
  - minimal `permissions:`; no `pull_request_target` checking out PR code
  - no `${{ github.event.* }}` interpolated directly into `run:` (pass it through `env:` instead)
  - actions are from trusted publishers and pinned to a version or SHA, never `@main`
  - no hard-coded or echoed secrets
  - Pages uploads only the site files
- **Hygiene:** no backups, logs, dumps, archives, large binaries, `node_modules/`, or other stray files.

Report each finding with its severity (critical, high, medium, low, or info), file and line, the risk, and a suggested fix. Then give a verdict:
- **PASS:** no critical or high findings. Continue.
- **FAIL:** **stop.** Push, edit the About, or enable Pages only after the user fixes each critical/high finding or explicitly accepts the risk for it. If a secret was ever committed, tell the user it must be rotated and removed from git history; deleting the file is not enough.

## 5. Push the code

1. Show `git status` and `git diff --stat`. Commit with `git add -A` and a clear message (e.g. "Add README and GitHub Pages workflow").
2. Remote: use `origin` if it already points to the requested repo. Otherwise add a new remote, or ask before changing `origin`'s URL.
3. Push with `git push -u <remote> <branch>`.
   - **Network error:** retry up to 4 times with backoff (2s, 4s, 8s, 16s).
   - **Rejected because the remote has new commits:** `git pull --no-rebase`, resolve, re-run the step 4 scan on anything new, then push.
   - **Authentication fails:** tell the user they need write access. Pushing `.github/workflows/` also needs a token with the `workflow` scope. Don't try to work around it.

## 6. Create or edit the repo About

The About is the repo's description, website, and topics. Get the current values first (`gh repo view owner/repo --json description,homepageUrl,repositoryTopics`). Keep anything the user set on purpose, and fill in or fix the rest:
- **Description:** one line, at most about 120 characters, e.g. "Single-file vanilla JS Kanban board demo for IT PMO task tracking: no build, no backend."
- **Website:** the GitHub Pages URL (or the custom domain).
- **Topics:** lowercase and hyphenated, about 5–8, e.g. `kanban`, `vanilla-javascript`, `html`, `github-pages`, `project-management`, `demo`. Don't use trademarked names as topics.

Apply it with `gh repo edit owner/repo --description "..." --homepage "..." --add-topic a,b,c`, or the equivalent API call. Without API access, give the user the exact values to paste in via the ⚙️ gear next to **About** on the repo page.

## 7. Enable GitHub Pages and check the deploy

1. **Set the Pages source to GitHub Actions.**
   - Try `gh api -X POST repos/owner/repo/pages -f build_type=workflow`; if Pages already exists, use `-X PUT` instead.
   - Otherwise tell the user: **Settings → Pages → Build and deployment → Source → GitHub Actions**.
   - If the repo is private, mention that Pages on a private repo needs a paid GitHub plan.
2. **Check the run.** The push in step 5 triggers it. Without `gh`, use the **Actions** tab, or start it there with **Run workflow** if it ran before Pages was enabled. Check its result with `gh run list --workflow pages.yml --limit 1` and `gh run watch`, or with the MCP `actions_*` tools.
3. **If it fails:** read the logs, fix the cause, run step 4 again, and push again.
4. **Custom domain:** tell the user to add a DNS `CNAME` record pointing to `<owner>.github.io` (or `A` records for an apex domain), and to enable **Enforce HTTPS** once the certificate is ready.

## 8. Report

Summarize:
- the security scan verdict and findings
- README: created or updated, and what changed
- the About values that were set
- the workflow file, the run result, and the live site URL
- the repo URL, branch, and pushed commit hash (`git rev-parse --short HEAD`)
- any manual steps the user still has to do

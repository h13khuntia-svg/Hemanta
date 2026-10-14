---
description: Security-scan the repository before anything is pushed or published to the internet
argument-hint: [optional: files or paths to focus on]
---

Run a security scan of this repository. This must pass before any code is pushed to GitHub, deployed to GitHub Pages, or otherwise uploaded.

Focus (may be empty — then scan everything that would be uploaded): $ARGUMENTS

Read-only: do not change any files during the scan. Report findings and let the user decide on fixes.

1. **Work out what would be uploaded.** Run `git status`, `git diff --stat` against the remote branch (if one exists), and `git ls-files`. Include untracked files that `git add -A` would pick up. Read every file in that set.

2. **Secrets and credentials** (in the files *and* in unpushed commit history — `git log -p @{u}..HEAD`, or the full history if there is no upstream):
   - API keys, tokens, passwords, private keys (`-----BEGIN ... PRIVATE KEY-----`), `ghp_`/`gho_`/`github_pat_`, `AKIA`, `sk-`, `xox[bp]-`, bearer tokens, connection strings, `.env` files, `*.pem`/`*.key`, `id_rsa`.
   - Personal data that shouldn't be public: email addresses, phone numbers, internal hostnames or IPs. The `FORMSUBMIT_ENDPOINT` email is expected — flag it as **info** so the user knows it will be public, not as a blocker.

3. **`index.html` code checks:**
   - **XSS:** every user-supplied or seed field that goes into `innerHTML`/template strings passes through `escapeHtml()`. Flag any `innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `eval`, `new Function`, or string `setTimeout`/`setInterval` that takes unescaped input.
   - **Network:** the only network call is the FormSubmit `fetch`, it sends only the task fields, and a failure only shows a toast. Flag any other `fetch`/`XMLHttpRequest`/`navigator.sendBeacon`/`WebSocket`/`<form action>`/image-beacon.
   - **External resources:** no `<script src>`, `<link href>`, `@import`, `url(...)`, or `<iframe>` pointing to `http(s)://` (CLAUDE.md forbids them; they're also a supply-chain risk).
   - **Storage:** no `localStorage`, `sessionStorage`, `indexedDB`, or `document.cookie`.
   - **Links:** any `target="_blank"` has `rel="noopener noreferrer"`.

4. **GitHub Actions workflows** (`.github/workflows/*`, if any):
   - `permissions:` is set and minimal; no `write-all`.
   - No `pull_request_target` that checks out PR code; no untrusted `${{ github.event.* }}` (titles, bodies, branch names) interpolated directly into `run:` scripts — they must go through `env:`.
   - Actions come from trusted publishers (`actions/*` or well-known orgs) and are pinned to a version tag or commit SHA, never `@main`/`@master`.
   - Secrets are referenced via `${{ secrets.* }}`, never hard-coded or echoed.
   - Pages deploys upload only the site files (`index.html`, `CNAME`, `.nojekyll`) — not `.git`, `.claude/`, `.github/`, or `CLAUDE.md`.

5. **Repository hygiene:** large binaries, archives, or unexpected executables; files that look like they don't belong (backups, `*.log`, dumps, `node_modules/`). Suggest a `.gitignore` entry if useful.

6. **Report.** List each finding with severity (**critical / high / medium / low / info**), file and line, what the risk is, and the suggested fix. End with a clear verdict:
   - **PASS** — nothing critical or high; safe to upload.
   - **FAIL** — at least one critical or high finding. Do **not** push, deploy, or upload anything until the user fixes it or explicitly accepts the risk for that specific finding.
   If a secret was already committed, say that deleting the file is not enough: the secret must be rotated and removed from git history before pushing.

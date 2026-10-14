---
description: Publish this board as a GitHub Pages site deployed by a GitHub Actions workflow, or update the existing setup
argument-hint: [optional: branch to deploy from, custom domain, or other changes]
---

Set up or update GitHub Pages for this repository, deployed through GitHub Actions.

What the user wants (may be empty): $ARGUMENTS

Follow these steps:

1. **Find out what's there.** Read `CLAUDE.md`, list the repo files, and read everything in `.github/workflows/`. Look for an existing Pages workflow (one using `actions/deploy-pages`) and any `CNAME` or `.nojekyll` file.

2. **Get the details.** If they weren't given in `$ARGUMENTS`, ask the user:
   - which branch should trigger a deploy (default: the repository's default branch, usually `main`)
   - whether they want a custom domain (default: no — the site will be at `https://<owner>.github.io/<repo>/`)
   Don't guess a custom domain.

3. **Create or edit the workflow** at `.github/workflows/pages.yml` (or edit the existing Pages workflow in place, changing only what was asked):
   - Triggers: `push` to the chosen branch, plus `workflow_dispatch` for manual runs.
   - `permissions:` `contents: read`, `pages: write`, `id-token: write`.
   - `concurrency:` group `pages`, `cancel-in-progress: false`.
   - **build** job on `ubuntu-latest`: `actions/checkout@v4` → `actions/configure-pages@v5` → copy only the site files into a `_site/` folder (`index.html`, plus `CNAME` and `.nojekyll` if present — never `.claude/`, `.github/`, `CLAUDE.md`, or other repo files) → `actions/upload-pages-artifact@v3` with `path: _site`.
   - **deploy** job: `needs: build`, `environment: { name: github-pages, url: ${{ steps.deployment.outputs.page_url }} }`, using `actions/deploy-pages@v4` with `id: deployment`.
   - If a validate workflow exists (see `/update-github-action`), or the user wants one, add a validate step or job before deploy so a broken `index.html` is never published.

4. **Keep the site as-is.** Pages serves `index.html` exactly as it is. Respect `CLAUDE.md`: no build step, framework, npm, `package.json`, extra CSS/JS files, or external resources. Add an empty `.nojekyll` file at the root only if needed to stop Jekyll processing.

5. **Custom domain (only if requested).** Create or update a `CNAME` file containing just the domain, and tell the user to add a DNS `CNAME` record pointing to `<owner>.github.io` (or `A` records for an apex domain) and to tick **Enforce HTTPS** once the certificate is issued.

6. **Check your work.** Validate the YAML (e.g. `python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" .github/workflows/pages.yml` if PyYAML is available) and confirm `index.html` is at the repo root.

7. **Tell the user the one manual step:** in the repo go to **Settings → Pages → Build and deployment → Source** and choose **GitHub Actions**. The first deploy runs on the next push to the chosen branch (or from the **Actions** tab via **Run workflow**).

8. **Show the result.** Summarize what you created or changed and give the expected site URL. Ask before committing; push only if the user asks, or suggest `/push-to-github`. Note that pushing files under `.github/workflows/` needs a GitHub token with the `workflow` scope, and that GitHub Pages on a private repo requires a paid GitHub plan.

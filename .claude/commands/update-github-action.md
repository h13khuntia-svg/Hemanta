---
description: Create a GitHub Actions workflow, or edit an existing one in .github/workflows/
argument-hint: [optional: what the workflow should do, e.g. "validate index.html" or "deploy to GitHub Pages"]
---

Create or edit a GitHub Actions workflow for this repository.

What the user wants (may be empty): $ARGUMENTS

Follow these steps:

1. **Find out what's there.** Read `CLAUDE.md` and list `.github/workflows/`. If workflows exist, read them all.

2. **Decide what to build.**
   - **The user described a goal:** do that.
   - **No goal given and no workflows exist:** ask the user which one they want, offering:
     - **Validate** (recommended): on push and pull request, check that `index.html` is sound — see step 3.
     - **Deploy to GitHub Pages:** hand off to the steps in `.claude/commands/update-github-pages.md`.
     - **Both.**
   - **No goal given and workflows exist:** summarize what they do and ask what to change.
   When editing, change only what was asked and keep the rest of the file as it is.

3. **The validate workflow** should run on `ubuntu-latest` and use only tools already on the runner (Node and Python are preinstalled). Steps:
   - `actions/checkout` (pinned to a major version, e.g. `@v4`).
   - Extract the inline `<script>` block from `index.html` and run `node --check` on it.
   - Check that every static `getElementById("...")` in the script has a matching `id="..."` in the HTML. Ids built by string concatenation (like `error-<field>` in `setFieldError`/`resetForm`) are dynamic — only check string literals.
   - Fail if the code breaks the hard rules in `CLAUDE.md`: any `localStorage`, `sessionStorage`, `indexedDB`, or `document.cookie`; any external `<script src=` or `<link href=` to `http(s)://`; or a `fetch(` call to anything other than the FormSubmit endpoint.
   Keep the checks inline in the workflow file (a short shell/Python/Node snippet). Don't add `package.json`, npm dependencies, a build step, or extra script files to the repo.

4. **Follow good workflow practice:**
   - Set minimal `permissions:` (e.g. `contents: read`; Pages needs `pages: write` and `id-token: write`).
   - Add `concurrency:` for deploy workflows so runs don't overlap.
   - Give each workflow and step a clear `name:`.
   - Never put secrets or email addresses in the workflow file; use `${{ secrets.NAME }}` if one is needed and tell the user to add it under **Settings → Secrets and variables → Actions**.

5. **Validate the YAML** before finishing (e.g. `python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <file>` if PyYAML is available), and run the validate checks locally against the current `index.html` so you know the workflow will pass. If it would fail, report why — don't loosen the checks just to make it pass.

6. **Show the result.** Summarize the workflow: triggers, jobs, and what each step checks or does. For a Pages deploy, tell the user to set **Settings → Pages → Source** to **GitHub Actions**. Ask before committing; push only if the user asks, or suggest `/push-to-github`. Before any push, run the security scan in `.claude/commands/security-scan.md` (`/push-to-github` does this automatically) and don't push if it fails. Note that pushing files under `.github/workflows/` needs a GitHub token with the `workflow` scope.

---
description: Commit and push this project's code to a GitHub repository the user specifies
argument-hint: [repo URL or owner/repo] [branch]
---

Push the code in this repository to GitHub.

Repo details provided by the user: $ARGUMENTS

Follow these steps:

1. **Get the repo details.** If `$ARGUMENTS` is empty or missing the repository, ask the user for:
   - the GitHub repository (full URL like `https://github.com/owner/repo` or `owner/repo`)
   - the branch to push to (default: the current branch, from `git branch --show-current`)
   - a commit message, if there are uncommitted changes (offer a sensible default based on the diff)
   Do not guess a repository. Wait for the user's answer before continuing.

2. **Check the working tree.** Run `git status` and `git diff --stat`. Show the user what will be committed. Never commit secrets, `.env` files, or credentials. If any appear, stop and ask.

3. **Security scan — required before uploading.** Run every step in `.claude/commands/security-scan.md` against what will be pushed. If the verdict is **FAIL**, stop: do not push until the user fixes the findings or explicitly accepts the risk for each one.

   Also sanity-check `index.html` (see CLAUDE.md):
   - the inline `<script>` is syntactically valid JavaScript
   - every static `getElementById("...")` call resolves to an `id="..."` in the HTML
   If either check fails, report it and ask whether to push anyway.

4. **Commit** any uncommitted changes with `git add -A` and the agreed commit message.

5. **Set the remote.**
   - If `origin` already points to the requested repository, use it.
   - Otherwise add or update a remote for it (`git remote add <name> <url>` or ask before running `git remote set-url origin <url>`). Normalize `owner/repo` to `https://github.com/owner/repo.git`.

6. **Push** with `git push -u <remote> <branch>`. On a network error, retry up to 4 times with exponential backoff (2s, 4s, 8s, 16s).
   - Never force-push unless the user explicitly asks for it.
   - If the push is rejected because the remote has newer commits, run `git pull --no-rebase <remote> <branch>`, resolve conflicts if needed, and push again.
   - If authentication fails, tell the user and explain they need GitHub access to that repo. Don't try to work around it.

7. **Report** the security scan verdict, the repository URL, branch, and pushed commit hash (`git rev-parse --short HEAD`). Do not open a pull request unless the user asks for one.

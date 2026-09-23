---
description: Security-scan, document and publish this project to a GitHub repo with a GitHub Pages site
argument-hint: <owner/repo | GitHub URL> [branch]
allowed-tools: Bash(git:*), Bash(gh:*), Bash(grep:*), Bash(find:*), Bash(ls:*), Bash(cat:*), Bash(du:*), Bash(file:*), Bash(gitleaks:*), Read, Edit, Write, mcp__playwright__browser_navigate, mcp__playwright__browser_resize, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_close
---

# Publish to GitHub

Publish this project to GitHub: README with a screenshot, repo "About" section, a GitHub Pages deploy via GitHub Actions and a security scan that **must pass before anything is pushed**.

Arguments: `$ARGUMENTS`

Follow the steps in order. Stop and report on any failure. Don't skip ahead.

## 0. Parse the target repo

- The first argument is the repo: `owner/repo`, `https://github.com/owner/repo(.git)` or `git@github.com:owner/repo.git`. Normalise it to `OWNER/REPO`.
- The optional second argument is the branch (default `main`).
- If no repo was given, show the current `git remote -v` and ask the user which repo to use. Don't guess.

## 1. Preflight

Run these and report the results:

- `git status --short`, `git branch --show-current`, `git remote -v`
- `gh auth status`. If you're not logged in, tell the user to run `gh auth login` themselves, then stop.
- `gh repo view OWNER/REPO --json name,visibility,description,homepageUrl,repositoryTopics,defaultBranchRef`
  - If the repo doesn't exist, ask whether to create it and whether it should be public or private. Only after a clear yes, run `gh repo create OWNER/REPO --public|--private --source . --remote origin`. (GitHub Pages on a free plan needs a public repo, so say that.)
- If `origin` points somewhere else, show both URLs and ask before running `git remote set-url origin ...`.

## 2. README

Read `CLAUDE.md` and the project files so the README is accurate. Create or update `README.md` with:

- Title and a one-paragraph summary of what the project is. For this repo, say clearly that it is an internal demo/training board and not an official system.
- **Live demo** link: `https://OWNER.github.io/REPO/` (lowercase owner).
- Features (short bullets).
- How to run locally (e.g. open `index.html`; no build step).
- Configuration (e.g. `FORMSUBMIT_ENDPOINT` and its one-time activation).
- Deployment: GitHub Pages via `.github/workflows/pages.yml`, triggered on push to the branch.
- Project structure and a short "Security notes" section (no persistence, escaped input, the only external endpoint).

Keep any correct content that is already there. Don't invent features. Show the user a summary of what changed.

## 2a. Screenshot for the README

Use the **Playwright MCP server** (configured in `.mcp.json`) to capture the site and embed it in the README.

1. Pick the page to capture:
   - If the live Pages site (`https://OWNER.github.io/REPO/`) is up and shows the current version, use it.
   - Otherwise (first publish, or the site is out of date) use the local file `file://<absolute path>/index.html`. If Playwright refuses `file://`, say so and use the live site, noting that the screenshot may be one deploy behind.
2. Call `browser_resize` with width 1440 and height 900, then `browser_navigate` to the page.
3. Call `browser_take_screenshot` with `filename: "docs/screenshot.png"`, `scale: "css"` (viewport only, not full page), then `browser_close`.
4. Open `docs/screenshot.png` and check it: the board, the four columns and the sample cards have rendered, and no error, blank page or Pages 404 is showing. If it looks wrong, fix the cause and retake it. Don't commit a broken image.
5. Make sure `README.md` embeds it just below the **Live demo** link, with descriptive alt text:
   `![Screenshot of the Kanban board showing the Backlog, In Progress, Blocked and Done columns with sample tasks](docs/screenshot.png)`
   Add `docs/screenshot.png` to the Project structure section if it isn't listed.
6. Make sure `.gitignore` contains `.playwright-mcp/` so Playwright's snapshots and logs are never committed.

The screenshot is only for the README. **Don't** add it to the "Assemble site" step in `pages.yml`, because the Pages site doesn't need it.

The security scan in step 4 covers `docs/screenshot.png` too. Look at the image for anything that shouldn't be public (a real email address, personal data, other browser tabs).

## 3. GitHub Pages workflow

Make sure `.github/workflows/pages.yml` exists and is correct. Create it or fix it so that it:

- runs on `push` to the target branch and on `workflow_dispatch`
- has minimal permissions: `contents: read`, `pages: write`, `id-token: write`
- has `concurrency: { group: pages, cancel-in-progress: false }`
- uses `actions/checkout@v4`, `actions/configure-pages@v5`, `actions/upload-pages-artifact@v3` and `actions/deploy-pages@v4`
- copies **only** the files the site needs into `_site` (plus `.nojekyll`), so that `CLAUDE.md`, `.claude/`, `.git*` and other files are never published

If there are new site files (images, css, js), add them to the "Assemble site" step.

## 4. Security scan (a gate: nothing is pushed unless this passes)

Scan **everything that would be pushed**: tracked files, files about to be committed and all commits not yet on the remote (`git log origin/BRANCH..HEAD` if the remote branch exists, otherwise the full history).

1. **Secrets**
   - If `gitleaks` is installed, run `gitleaks detect --source . --redact -v` (it scans history too).
   - Always also grep the working tree and `git log -p` for: private keys (`-----BEGIN .*PRIVATE KEY-----`), `AKIA[0-9A-Z]{16}`, `ghp_`/`gho_`/`github_pat_`, `xox[baprs]-`, `sk-[A-Za-z0-9]{20,}`, `sk-ant-`, `AIza[0-9A-Za-z_-]{35}`, `password\s*[:=]`, `secret\s*[:=]`, `token\s*[:=]`, `api[_-]?key\s*[:=]` and connection strings (`://[^/\s:]+:[^@\s]+@`).
2. **Sensitive files**: flag any tracked or staged `.env*`, `*.pem`, `*.key`, `*.p12`, `id_rsa*`, `*.sqlite`/`*.db`, credentials/config dumps, `.DS_Store`, and anything larger than 5 MB. Offer to add a `.gitignore` for the junk ones.
3. **Personal data**: list any email addresses, phone numbers or internal hostnames/IPs found in files that will be published to the Pages site. A real email in `FORMSUBMIT_ENDPOINT` becomes publicly visible, so flag it as a warning and let the user decide.
4. **Client-side code review of the published HTML/JS**:
   - user input rendered through `innerHTML` without `escapeHtml()`
   - `eval`, `new Function`, `document.write`, or `setTimeout` with a string argument
   - any external URL other than the approved endpoint, `<script src>` from a CDN, and forms posting over `http://`
   - `localStorage`/`sessionStorage`/`indexedDB`/`document.cookie` (this breaks the project's no-persistence rule)
   - `target="_blank"` without `rel="noopener noreferrer"`
5. **Workflow hardening**: least-privilege `permissions`, no `pull_request_target`, no secrets echoed to logs, and actions pinned to major versions or SHAs from trusted owners (`actions/*`).
6. **Brand/compliance** (from `CLAUDE.md`): no real logos, trademarks or imitation of an official system.

Report the findings as a table: **Severity (High / Medium / Low) · File:line · Finding · Fix**.

- **High** findings (any real secret, private key, credential or XSS sink with user data) mean you **stop**. Propose fixes and don't push. If the secret is already in history, say it must be rotated and that the history needs rewriting. Don't rewrite history without explicit approval.
- For **Medium/Low** findings, list them and ask whether to fix them first or continue.
- If nothing is found, say "Security scan passed" and list what was checked.

## 5. Commit and push (needs confirmation)

- Show `git status` and `git diff --stat`, and propose a commit message.
- **Ask the user to confirm** before committing and pushing. Pushing publishes the code to the internet.
- After a clear yes: stage the specific files by name (not `git add -A`; include `docs/screenshot.png`, `.gitignore` and `.mcp.json` if they changed), commit, then run `git push -u origin BRANCH`. Never force-push unless the user asks for it explicitly.

## 6. Repo "About" section

Propose a description (one line, under 350 characters), the homepage URL (the Pages URL) and 3–8 topics (lowercase, hyphenated, e.g. `kanban`, `vanilla-js`, `github-pages`, `demo`). After the user agrees, run:

```
gh repo edit OWNER/REPO --description "..." --homepage "https://owner.github.io/REPO/" --add-topic kanban --add-topic vanilla-js ...
```

## 7. Enable GitHub Pages (source = GitHub Actions)

- Check with `gh api repos/OWNER/REPO/pages`.
  - If you get 404: `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`
  - If it exists with `build_type` ≠ `workflow`: `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`
- If the API call fails (for example because of permissions or a private repo on the free plan), tell the user to set **Settings → Pages → Source → GitHub Actions** by hand.
- Trigger or watch the deploy: `gh run list --workflow pages.yml -L 1`, then `gh run watch <id> --exit-status`. If the push happened before Pages was enabled and the run failed, run `gh workflow run pages.yml --ref BRANCH` and watch that run.

## 8. Report

End with a short summary:

- Repo URL and branch pushed (commit SHA)
- Security scan result (passed, or what was fixed or accepted)
- README changes (including whether the screenshot was taken from the live or local site) and About section values
- Pages workflow status and the **live URL**
- Anything the user still needs to do by hand (e.g. rotate a secret, activate the FormSubmit email, set the Pages source)

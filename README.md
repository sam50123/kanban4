# kanban4: IT PMO Kanban Board (demo)

A single-page Kanban board for tracking IT project tasks across four columns: Backlog, In Progress, Blocked and Done. It's written in plain HTML, CSS and JavaScript in one file, with no build step and no dependencies.

> **Disclaimer:** This is an internal demo/training project. It is **not** an official UOB system and isn't affiliated with or endorsed by UOB. It uses a plain text wordmark only, with no real logos or trademarks.

**Live demo:** https://sam50123.github.io/kanban4/

## Features

- Four-column board with per-column task counts and a header summary (total, per status and overdue)
- Drag and drop between columns, plus a keyboard-accessible **Move ▸** menu on each card
- **Add Task** dialog with inline validation (title, project, category, assignee, priority, due date, status)
- Filters by project, assignee (text search) and priority
- Overdue badge on any task past its due date that isn't Done
- Inline **Delete? Yes / No** confirmation (Escape cancels)
- Optional email notification for each new task via [FormSubmit](https://formsubmit.co)
- Responsive layout, visible focus styles, screen-reader labels and support for reduced motion

## Run locally

No install or server is needed. Clone the repo and open `index.html` in a browser:

```bash
git clone https://github.com/sam50123/kanban4.git
open kanban4/index.html
```

The board loads with 8 sample tasks. Tasks are kept **in memory only**, so refreshing the page resets the board. This is by design.

## Configuration: email notifications

New-task notifications go to the address set in `FORMSUBMIT_ENDPOINT` at the top of the `<script>` block in `index.html`:

```js
const FORMSUBMIT_ENDPOINT = "https://formsubmit.co/ajax/YOUR_EMAIL@example.com";
```

- While the placeholder is set, no request is sent and each new task shows a "email notification failed" warning. The card is still added.
- A new address needs **one-time activation**: the first submission sends a confirmation email, and nothing is delivered until you click its link.
- The address is visible to anyone who views the page source on the public site. Use a shared or alias mailbox rather than a personal one.

## Deployment

The GitHub Actions workflow [`.github/workflows/pages.yml`](.github/workflows/pages.yml) publishes the site to GitHub Pages on every push to `main`. You can also run it by hand from the Actions tab.

- The repo's **Settings → Pages → Source** must be set to **GitHub Actions**.
- Only `index.html` is published. If you add files the site needs, copy them into `_site` in the workflow's "Assemble site" step.

## Project structure

```
index.html                    # the whole app: markup, one <style>, one <script>
.github/workflows/pages.yml   # GitHub Pages deploy
.claude/commands/publish.md   # Claude Code /publish command (scan → document → push → Pages)
CLAUDE.md                     # guidance for Claude Code
```

## Security notes

- **No persistence:** no localStorage, sessionStorage, IndexedDB or cookies.
- All user-entered text is HTML-escaped before rendering. There's no `eval` and no inline event handlers.
- The page loads no third-party scripts, fonts or images. The only external request is the FormSubmit POST.
- The deploy workflow uses least-privilege permissions (`contents: read`, `pages: write`, `id-token: write`).

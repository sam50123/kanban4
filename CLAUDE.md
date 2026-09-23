# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file internal demo/training Kanban board for "UOB IT PMO" (`index.html`). It is not an official UOB system: use a neutral text wordmark and a corporate blue palette only. No real logos, trademarks or imitation of official systems.

There are two standalone versions of the board, each a complete single file with the same script architecture: `index.html` (the original design) and `redesign.html` (a redesign with a light header, a status donut showing % complete, and restyled columns and cards). Changes to board behaviour usually need to be made in both.

## Running

There is no build, no package manager and no test suite. Open `index.html` directly in a browser (double-click / `file://`); no server is required. To verify changes, load the page in the browser pane and exercise it with the JS console (e.g. inspect `state.tasks`, dispatch `change` on a `.move-select`, dispatch `DragEvent`s on `.card`/`.column`).

## Hard constraints (from the original brief — keep them)

- Vanilla HTML/CSS/JS in **one file**: markup, one `<style>` block, one `<script>` block. No frameworks, CDNs, web fonts, image files or bundlers. Icons are Unicode/inline SVG and fonts use the system stack.
- **No persistence**: no localStorage, sessionStorage, IndexedDB or cookies. A refresh resets to the seeded demo data by design, and the header note says so.
- No `alert()`/`confirm()`. Validation errors are inline and delete confirmation is an inline "Delete? Yes / No" in the card.
- No `!important`. Palette and spacing are CSS custom properties on `:root`.
- The only backend is FormSubmit's AJAX endpoint. The only external URL in the file is `FORMSUBMIT_ENDPOINT`.

## Architecture (inside `<script>`)

- **Single source of truth**: `state = { tasks, filters, pendingDeleteId }`. The UI is always re-rendered from state. Don't mutate card DOM outside `renderBoard()`.
- **Render path**: `renderBoard()` builds all four columns via `applyFilters()` and `renderCard()` using HTML strings, then calls `renderSummary()`. Column count badges reflect the *filtered* view, while the header summary reflects *all* tasks. Every user-supplied string must pass through `escapeHtml()`.
- **Mutations**: `addTask()`, `moveTask()` and `deleteTask()` update `state` and then call `renderBoard()`. Because a re-render replaces the DOM, callers restore keyboard focus with `focusAfterRender()`.
- **Events** are delegated once on `#board` (dragstart/dragover/dragleave/drop/dragend, `change` for the "Move ▸" select, `click` for delete actions, and Escape to cancel a delete). Don't attach listeners to individual cards.
- **Task IDs** are `UOB-ITPM-####` from the `nextIdNumber` counter. The seed data uses IDs 1–8, and seed due dates are relative to today (`addDays()`) so the overdue badges always show.
- **Dates** are local-date ISO strings (`todayISO()`/`toISODate()`, not UTC) compared as strings. `isOverdue` = due date is before today and status ≠ Done.
- **Add Task flow** (`handleSubmit`): `validateForm()` → `addTask()` (optimistic) → reset form → success toast → `notifyNewTask()` in try/catch with the button in a "Sending…" state. On failure the card stays and a warning toast appears. The modal stays open after submit.

## FormSubmit

- To change the recipient email, edit `FORMSUBMIT_ENDPOINT` at the top of the script; it's the only place the address appears.
- While the placeholder `YOUR_EMAIL@example.com` is set, `notifyNewTask()` throws before sending any request, so every add shows the "email notification failed" warning.
- A new address needs one-time activation: the first submission sends a confirmation email, and nothing is delivered until its link is clicked.

## Deployment

`.github/workflows/pages.yml` publishes `index.html` and `redesign.html` (only) to GitHub Pages on every push to `main`. The repo's Pages source must be set to "GitHub Actions". If you add files the site needs, copy them into `_site` in the workflow's "Assemble site" step.

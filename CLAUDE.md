# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-file IT PMO Kanban board (`index.html`) for a fictitious bank, used as an internal demo/training app. All HTML, CSS and JS live in that one file. There is no build step, package manager, linter or test suite. Don't add any of these unless asked.

To run it, open `index.html` in a browser, or serve the folder (for example `python -m http.server`) and load the page.

## Architecture (inside `index.html`)

- **State is in memory only.** The single `state` object (`tasks`, `filters`, `ui`) is the source of truth. A page refresh re-seeds the board from `seedTasks()`. Seed due dates are relative to today (`addDays(n)`), so the overdue examples always show as overdue. There is no persistence layer, and the demo note in the header promises that.
- **Mutate, then re-render everything.** Each action changes `state` and then calls `renderBoard()`. That function rebuilds every column from HTML template strings and also calls `renderSummary()` and `restoreFocus()`. There is no incremental DOM patching.
- **Escaping.** Every user-supplied value put into template strings must go through `escapeHtml()`. Toasts use `textContent`.
- **Event delegation.** Card buttons carry `data-action` and `data-id`, and one `click` listener on `#board` (`handleBoardClick`) dispatches on the action. To add a card action, add a button with a new `data-action` in `renderCard()` and a matching `case` in that switch.
- **Inline card UI state.** `state.ui.pendingDeleteId` (the "Delete? Yes/No" row) and `state.ui.moveMenuId` (the keyboard-accessible "Move ▸" menu) decide which footer `renderCard()` draws. Only one can be open at a time, and Escape closes it.
- **Focus management.** The board re-renders on every change, so handlers set `state.ui.focus = { id, action }` before rendering. `restoreFocus()` then focuses that card's `[data-action]` button, falling back to the card and then to the "+ Add Task" button. Keep this pattern so keyboard and screen-reader users don't lose their place.
- **Drag and drop** uses the native HTML5 API, delegated on `#board` (`bindDragAndDrop`). A drop calls the same `moveTask()` as the Move menu.
- **Reference data.** `STATUSES` (which are also the columns), `PROJECTS`, `CATEGORIES` and `PRIORITIES` are constants near the top of the script. They fill the selects in `init()` and act as allow-lists in `validateForm()`. CSS has per-status rules (`.column[data-status="…"]`) and per-priority rules (`.card.prio-<slug>`), so renaming a value also means updating that CSS.
- **Add Task flow.** `handleSubmit` runs `readForm` → `validateForm` → `showFieldErrors`. It then adds the card straight away, before the email, and calls `notifyNewTask()` in the background. If the email fails, the user only sees a warning toast and the card stays. `state.ui.sending` disables the submit and opener buttons while a request is in flight.

## FormSubmit email integration

New tasks are POSTed as JSON to `FORMSUBMIT_ENDPOINT` (formsubmit.co AJAX). The config block at the top of the script holds this constant along with `ID_PREFIX` and `EMAIL_SUBJECT_TAG`. The address is a placeholder, `YOUR_EMAIL@example.com`.

FormSubmit needs a one-time activation per address: the first submission sends a confirmation email, and nothing is delivered until its link is clicked. FormSubmit can also return HTTP 200 with `success: "false"`, and `notifyNewTask` treats that response as a failure.

## Conventions

- Styling uses the CSS custom properties defined in `:root` (palette, spacing scale `--space-1..6`, radii, shadows). Use these tokens rather than hard-coded values. The CSS and JS are split into numbered banner-comment sections; keep new code in the matching section.
- Responsive breakpoints are 1100px (two columns) and 767px (one column, stacked form). The page also respects `prefers-reduced-motion`.

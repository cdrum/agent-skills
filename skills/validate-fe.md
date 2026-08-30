---
name: validate-fe
description: Validate front-end changes in a real Chrome tab — starts the dev server if needed, walks the affected routes, checks console errors, light and dark rendering, and the specific behavior that changed, then reports a verdict.
---

# Validate Front-end Changes

Use after a front-end change to confirm the fix actually works in the browser and nothing around it regressed. Both Wendways and OtterFin are Next.js App Router apps whose conventions this skill assumes.

This drives **your real Chrome** through the Claude in Chrome extension, not a headless browser. That is the point: your existing logged-in session is reused, so no test credentials are needed. It also means anything you do is done to your real dev environment — see the guard rails in Step 5.

---

## Step 0 — Load the browser tooling

1. Invoke the `claude-in-chrome` skill. Do this **before** any `mcp__claude-in-chrome__*` call.
2. If those tools are deferred, load them in **one** `ToolSearch` call, not one per tool:

   ```
   ToolSearch: select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__read_console_messages,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__find,mcp__claude-in-chrome__tabs_close_mcp
   ```
3. Call `tabs_context_mcp` to see what is already open, then work in a **new tab** (`tabs_create_mcp`) unless the user asks you to use one of theirs. Never reuse a tab id from an earlier session.

If the extension does not respond, say so and stop — do not fall back to curl-ing pages and calling that a front-end validation.

---

## Step 1 — Ask what to validate

Use `AskUserQuestion` — do not print a list and wait:

```
AskUserQuestion({
  questions: [{
    question: "What would you like to validate?",
    header: "Mode",
    multiSelect: false,
    options: [
      { label: "Issue-specific", description: "Walk the routes touched by the current branch and verify the change" },
      { label: "Smoke test", description: "Quick pass over the app's key routes for crashes and console errors" },
      { label: "Interactive flow", description: "Walk a user journey — form, dialog, filter, keyboard path" },
      { label: "Custom", description: "Describe a specific check to run" }
    ]
  }]
})
```

Use the answer and do not ask again.

---

## Step 2 — Get the dev server up

Resolve the port from the repo rather than assuming 3000 — neither app uses it:

| Repo | Dev URL | Script |
|---|---|---|
| `wendways` | `http://localhost:4020` | `next dev -p 4020` |
| `otterfin` | `http://localhost:4010` | `next dev -p ${PORT:-4010}` in `apps/community/package.json` |

If the repo is neither, read the `dev` script out of `package.json` and use that port.

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>
```

A `200`, `302`, or `307` means it is up — proceed. Otherwise start it in the background (`pnpm dev`), tell the user you are doing so, and poll the same curl until it answers rather than sleeping a fixed interval.

Both apps need PostgreSQL. If the server starts but pages 500 on data, check `docker compose ps` before blaming the change — a stopped database looks exactly like a broken query, and reporting it as a regression is worse than reporting nothing.

---

## Step 3 — Establish context (issue-specific mode)

```bash
git rev-parse --abbrev-ref HEAD
git diff --stat $(git merge-base HEAD origin/main)..HEAD
```

Pull the Linear id (`WW-…` / `OF-…`) from the branch name and read the issue if you need the acceptance criteria. Map changed files to routes:

- **Wendways** — `src/app/<route>/…` → `http://localhost:4020/<route>`.
- **OtterFin** — `apps/community/app/<route>/…` → `http://localhost:4010/<route>`. A change under `packages/ui/` has no route of its own: grep `apps/community/app` for the component's importers and visit every route that renders it.

Known routes, for when the diff does not name one:

- **Wendways:** `/`, `/trips`, `/plans`, `/map`, `/stats`, `/history`, `/wishlist`, `/import`, `/captures`, `/data-sources`, `/settings`
- **OtterFin:** `/dashboard`, `/dashboard/transactions`, `/dashboard/accounts`, `/dashboard/budget`, `/dashboard/reports`, `/dashboard/categories`, `/dashboard/tags`, `/dashboard/import`, `/dashboard/settings`

---

## Step 4 — Run the validation

Both apps require a session. If a navigation lands on `/sign-in` (Wendways) or `/auth/signin` (OtterFin), the browser profile is logged out — tell the user and let them log in rather than trying to authenticate on their behalf.

For every route visited:

1. `navigate` to the URL.
2. `read_page` for the accessibility tree — use this, not a screenshot, to assert on text, table cells, column headers, and control labels. Screenshots (`computer` with a screenshot action) are for layout and for showing the user what you saw.
3. `read_console_messages` — filter with the `pattern` parameter (e.g. `error|Error|Warning`) instead of pulling the whole log.
4. Note anything visibly broken: error boundaries, empty states where data was expected, overlapping or unstyled content.

**Mode: issue-specific.** Visit each affected route, then assert the specific behavior the issue describes — the changed value renders, the new control appears and responds, the bug's reproduction steps no longer reproduce. Walk the actual reproduction from the issue where there is one; a page that merely loads is not evidence the fix works.

**Mode: smoke test.** Walk the route list above in order, applying the four checks to each. Report the first crash but keep going — one broken route should not hide the rest.

**Mode: interactive flow.** Ask the user to describe the journey, then drive it with `computer` (click, type) and `form_input` for form fields, re-reading the page after each step. Check console errors at the end of the sequence, not after every keystroke.

**Mode: custom.** Plan a sequence from the user's description and finish with a console check.

### Dark mode

Both projects require `dark:` variants on every component from the start, and both drive them with `next-themes`' `.dark` class. Whenever the change touched UI, check both themes: use the app's own theme control if it has one, otherwise toggle the class directly with `javascript_tool`:

```js
document.documentElement.classList.toggle('dark')
```

Screenshot both. Unreadable contrast, an invisible border, or a stray light-mode background is a real finding — it is the single most common miss in these repos.

---

## Step 5 — Guard rails

- **Never trigger a native dialog.** `alert`, `confirm`, and `prompt` block the extension and kill the session. That means: do not click Delete buttons, "Remove account", or anything with a confirmation step unless the user explicitly asks and accepts the risk.
- **This is real data.** Both apps run against your own dev database — OtterFin holds real household finances, Wendways real trips. Create test records if you must, and say what you created; do not edit or delete existing rows to make a check convenient.
- **Use `console.log` plus `read_console_messages` for debugging**, never `alert`.
- Stop and ask after two or three failed attempts at the same interaction rather than trying variations of the same click.

---

## Step 6 — Report

```
## Validation Results

**Mode:** <mode>
**Routes checked:** <list>

### Console
<errors and warnings found, or "clean">

### Rendering
<what the screenshots show, light and dark>

### Behavior
<what was asserted, and what the page actually did>

### Verdict
✅ Confirmed — <what specifically was verified>
⚠️ Issues found — <what broke, where, and how to reproduce>
```

Say what you actually checked, not what the mode was supposed to cover. If a route was skipped because the server or database was down, list it as skipped rather than passing it.

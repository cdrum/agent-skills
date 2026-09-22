---
name: start-issue
description: Start work on a Linear issue — verifies the issue exists, assigns it to you, moves it to In Progress, and creates a correctly prefixed git branch off the repo's default branch.
---

# Start a Linear Issue

`/start-issue <linear-issue-id>`, e.g. `/start-issue WW-342`

Linear calls go to whichever Linear MCP server this session has connected: `get_issue`, `save_issue`, `list_issue_statuses`, `list_issue_labels`, called by whatever names this session exposes. If no Linear server is connected, say so and stop.

---

## Step 1 — Fetch the issue

`get_issue { "id": "<ID>", "includeRelations": true }`. If it isn't found, stop and say so. Note the team, state, assignee, labels, `gitBranchName`, and `blockedBy`. If the issue is blocked by something that isn't Done, say so and ask whether to start anyway.

## Step 2 — Assign and move to In Progress

Make one call, sending only the fields that need to change (skip it if neither does):

`save_issue { "id": "<ID>", "assignee": "me", "state": "In Progress" }`

If the `state` name is rejected, run `list_issue_statuses { "team": "<TEAM_KEY>" }` and use the status whose type is `started`.

## Step 3 — Pick the branch prefix

Match the issue's labels case-insensitively, which covers both `Bug` and `type:bug` styles:

| Label contains | Prefix |
|---|---|
| `bug` | `fix/` |
| `feature` | `feature/` |
| `chore` or `setup` | `chore/` |
| `hotfix` | `hotfix/` |

If exactly one label matches, use it. If none or several match, ask the user to pick `feature`, `fix`, `chore`, or `hotfix`, then add the matching label to the issue (find the exact name with `list_issue_labels`). `labels` **replaces** the whole set, so send the existing labels plus the new one.

## Step 4 — Create the branch

Resolve the default branch (`git symbolic-ref --short refs/remotes/origin/HEAD | sed 's|^origin/||'`, falling back to `git remote show origin`, and asking if both fail). Then:

```bash
git checkout <base> && git pull origin <base>
git checkout -b <prefix><gitBranchName without Linear's leading "username/">
```

Example: `feature/ww-342-venue-typeahead`. If the repo's `AGENTS.md` / `CLAUDE.md` gives a different convention (OtterFin also allows `plugin/`), follow that instead. If any git command fails, report it and stop.

## Step 5 — Report

Start with one plain sentence, e.g. "You're set up on WW-342 (venue typeahead): it's assigned to you, marked In Progress, and you're on `feature/ww-342-venue-typeahead`." Then list only what actually changed, such as a label that was added.

---
name: start-issue
description: Start work on a Linear issue — verifies the issue exists, assigns it to you, moves it to In Progress, and creates a correctly prefixed git branch off the repo's default branch.
---

# Start a Linear Issue

## Usage

`/start-issue <linear-issue-id>`

Examples: `/start-issue WW-342`, `/start-issue OF-91`

---

## Tools

Linear calls go to whatever Linear MCP server this session has connected. Use the server's own tool names: `get_issue`, `save_issue`, `list_issue_statuses`, `list_issue_labels`. A harness prefixes those differently — Claude Code exposes `mcp__claude_ai_Linear__get_issue`, Cursor exposes a namespaced tool — so discover the Linear tools in this session and call the one whose name ends in the tool below. Do not paste a prefixed name from another harness. If no Linear server is connected, say so and stop.

---

## Step 1 — Fetch and verify the issue

```
get_issue  { "id": "<ISSUE_ID>", "includeRelations": true }
```

If the issue is not found, stop and tell the user.

Note the `team`, `state`, `assignee`, `labels`, and `gitBranchName` from the response. `includeRelations` returns `blockedBy` — if the issue is blocked by an issue that is not yet Done, say so and ask whether to start it anyway before doing anything else.

---

## Step 2 — Assign it to yourself and move it to In Progress

Both are a single `save_issue` call. `assignee` accepts the literal string `"me"`, and `state` accepts a status **name**, so there is no need to look up a user ID or a workflow state ID:

```
save_issue  { "id": "<ISSUE_ID>", "assignee": "me", "state": "In Progress" }
```

Only send the fields that actually need changing:

- If the issue is already assigned to you, omit `assignee` and say nothing.
- If it is already In Progress, omit `state` and say nothing.
- If both are already correct, skip the call entirely.

Report only what you changed (e.g. "Assigned to you, moved to In Progress.").

If `state: "In Progress"` is rejected, the team's status is named something else — list the team's statuses and pick the one whose type is `started`:

```
list_issue_statuses  { "team": "<TEAM_KEY>" }
```

---

## Step 3 — Determine the branch type prefix

Read the issue's labels and map to a prefix. Match case-insensitively on the label name, so both plain (`Bug`, `Feature`) and namespaced (`type:feature`, `type:chore`, `type:setup`) taxonomies resolve:

| Label contains | Prefix |
|---|---|
| `bug` | `fix/` |
| `feature` | `feature/` |
| `chore` or `setup` | `chore/` |
| `hotfix` | `hotfix/` |

- **Exactly one match** → use it, do not ask.
- **No match or several** → ask the user to choose one of `feature`, `fix`, `chore`, `hotfix`. Do not accept another answer.

**Missing label:** if you had to ask, apply the matching label after the user answers. Look up the team's labels to find the exact name in use:

```
list_issue_labels  { "team": "<TEAM_KEY>" }
```

Then save it. **`labels` replaces the entire label set**, so send the issue's existing label names plus the new one — sending only the new name silently drops the others:

```
save_issue  { "id": "<ISSUE_ID>", "labels": ["<existing>", "...", "<new>"] }
```

Tell the user which label you added.

---

## Step 4 — Sync the base branch and create the branch

Derive the default branch from the remote rather than assuming one:

```bash
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||'
```

If that prints nothing (the ref is not set locally), run `git remote show origin | sed -n 's/.*HEAD branch: //p'`. If it still cannot be resolved, ask the user which branch to base off. Do not fall back to a guess.

Check out and update it:

```bash
git checkout <base-branch> && git pull origin <base-branch>
```

If either command fails, report the error and stop.

Then build the branch name from the Linear `gitBranchName`, **stripping the leading `<username>/` segment Linear adds**, and prepend the prefix from Step 3:

```
feature/ww-342-venue-typeahead-and-premium-upsell
fix/of-91-arrow-key-navigation-in-dropdowns
```

```bash
git checkout -b <branch-name>
```

Repo conventions this matches:

| Repo | Convention |
|---|---|
| `wendways` | `<type>/ww-<n>-<slug>` with `<type>` from the issue's `type:` label — the repo's agent instructions say to ignore Linear's suggested `<username>/…` form |
| `otterfin`, `otterfin-cloud` | `feature/`, `fix/`, or `plugin/` plus the issue id — see `AGENTS.md` § Git & PR Conventions |

If the repo's own `CLAUDE.md` or `AGENTS.md` states a different convention, that file wins over this table.

Confirm the branch was created.

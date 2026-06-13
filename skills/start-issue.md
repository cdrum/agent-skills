---
name: start-issue
description: Start work on a Linear issue — verifies the issue exists and creates a prefixed git branch.
---

# Start a Linear Issue

## Usage

`/start-issue <linear-issue-id>`

Example: `/start-issue DMP-1234`

## Instructions

### 1. Fetch and verify the issue

Use the Linear MCP to retrieve the issue:

```
mcp__claude_ai_Linear__get_issue  { "id": "<ISSUE_ID>" }
```

If the issue is not found, stop and tell the user.

### 1a. Assign the issue to yourself and move it to In Progress

Fetch the currently authenticated Linear user:

```
mcp__claude_ai_Linear__get_viewer  {}
```

**Assignee:** If the issue's `assignee.id` is missing or does not match the viewer's `id`, assign the issue to yourself:

```
mcp__claude_ai_Linear__update_issue  { "id": "<ISSUE_ID>", "assigneeId": "<VIEWER_ID>" }
```

Tell the user if you assigned it (e.g. "Assigned to you."). If already assigned to you, say nothing.

**Status:** If the issue's `state.name` is not already `"In Progress"`, fetch the team's workflow states to get the correct state ID:

```
mcp__claude_ai_Linear__get_workflow_states  { "teamId": "<TEAM_ID>" }
```

Find the state whose `name` is `"In Progress"` and update the issue:

```
mcp__claude_ai_Linear__update_issue  { "id": "<ISSUE_ID>", "stateId": "<IN_PROGRESS_STATE_ID>" }
```

Tell the user if you moved it (e.g. "Status set to In Progress."). If it was already In Progress, say nothing.

### 2. Determine the branch type prefix

Check the labels on the issue and map to a prefix:

| Label | Prefix |
|---|---|
| `Bug` | `bugfix/` |
| `Feature` | `feature/` |
| `Hotfix` | `hotfix/` |

- **Exactly one match** → use it, no need to ask
- **Multiple matches or no match** → ask the user to choose one of: `feature`, `bugfix`, `hotfix`, `release`

Do not accept any other answer.

**Missing label:** If no matching label was found on the issue (i.e. you had to ask the user), apply the corresponding label after the user answers. Map the chosen type back to its label name (`feature` → `Feature`, `bugfix` → `Bug`, `hotfix` → `Hotfix`), fetch the team's labels to get the label ID, then add it:

```
mcp__claude_ai_Linear__get_labels  { "teamId": "<TEAM_ID>" }
```

```
mcp__claude_ai_Linear__update_issue  { "id": "<ISSUE_ID>", "labelIds": [<EXISTING_LABEL_IDS..., "<NEW_LABEL_ID>"] }
```

Preserve any labels already on the issue. Tell the user (e.g. "Added label Feature.").

### 3. Sync the base branch and create the branch

Determine the correct base branch for the current repo:

| Repo | Base branch |
|------|-------------|
| `dmp` | `dev` |
| `admin` | `dev` |
| `altpe` | `develop` |

If the current repo is not listed above, default to `dev`. If you are unsure which repo you are in, ask the user: **"Which base branch should I branch from? (dev / develop / main)"**

Switch to the base branch and pull the latest changes:

```bash
git checkout <base-branch> && git pull origin <base-branch>
```

If this fails, report the error and stop.

Then create the new branch from the updated base branch. Strip any `username/` prefix from the Linear-suggested `branchName`, then prepend the chosen prefix:

```
feature/dmp-1234-my-issue-title
bugfix/fe-456-broken-export-flow
```

```bash
git checkout -b <branch-name>
```

Confirm the branch was created.

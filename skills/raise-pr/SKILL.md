---
name: raise-pr
description: Raise a GitHub pull request from the current branch — checks for uncommitted changes and a CHANGELOG entry, syncs with remote, then creates a PR against the repo's default branch with a contextual title and engineer-focused description.
---

# Raise a Pull Request

## Usage

`/raise-pr`

Raises a pull request for the current branch against the default base branch. Works through pre-flight checks, pushes if needed, then creates the PR on GitHub.

---

## Step 0 — Resolve the base branch

Derive the repo's default branch from the remote rather than assuming one:

```bash
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||'
```

If that prints nothing (the ref is not set locally), fall back to `git remote show origin | sed -n 's/.*HEAD branch: //p'`. If it still cannot be resolved, ask the user which branch to target — do not guess.

Every personal repo (`wendways`, `otterfin`, `otterfin-cloud`, and the site/docs repos) uses `main`. Refer to the resolved value as `<base>` everywhere below and never hardcode it, so this skill keeps working in a repo that bases off `dev` or `develop`.

---

## Step 1 — Check for uncommitted changes

Run `git status --short` to check the working tree.

**If there are staged but uncommitted changes**, stop immediately and tell the user:

> "You have staged changes that haven't been committed. Please commit or stash them before raising a PR."

Do not proceed until the staged changes are resolved.

**If there are only unstaged changes** (modified or untracked files), warn the user:

> "You have unstaged changes that won't be included in this PR:
> [list the files]
>
> Continue without including these changes? (y/n)"

If the user says no (or stop / abort / anything negative), stop here.
If the user says yes (or continue / proceed), move to Step 2.

If the working tree is clean, move straight to Step 1b.

---

## Step 1b — Check the changelog covers this branch

Both Wendways and OtterFin keep a single `CHANGELOG.md` at the repo root in Keep a Changelog format, and both require an entry for any user-visible or developer-relevant change (OtterFin states this in `AGENTS.md` § Git & PR Conventions). If there is no `CHANGELOG.md` at the repo root, skip this step silently.

Check whether this branch touched it:

```bash
git diff --stat $(git merge-base HEAD origin/<base>)..HEAD -- CHANGELOG.md
```

If the diff is empty, stop and tell the user:

> "This branch adds no `CHANGELOG.md` entry. Add one (`/commit` writes it), or confirm you want to raise the PR without it."

- **skip / yes** → proceed to Step 2.
- **no / stop** → stop here.

Do not write the entry yourself here — this skill raises PRs, it does not author changelog copy.

If the branch does touch `CHANGELOG.md`, read the added lines and check they sit under `## [Unreleased]`, name the branch's Linear issue id (`WW-…` / `OF-…`), and read as finished prose rather than a placeholder. Mention any of that in passing; do not block on it.

---

## Step 2 — Identify the current branch and remote sync status

Run:
```bash
git rev-parse --abbrev-ref HEAD
```
Note the branch name. If it is `main` or `master`, warn the user that raising a PR from that branch is unusual and ask them to confirm before continuing. `dev` and `develop` are valid working branches — no warning needed.

Then check whether local commits have been pushed:
```bash
git log --oneline @{u}..HEAD 2>/dev/null || git log --oneline HEAD
```

If there are local commits not yet on the remote (or no upstream is set), ask the user:

> "Your branch has unpushed commits. Push now and continue? (y/n)"

If the user says no, stop here.

If the user says yes, push the branch:
```bash
git push -u origin HEAD
```

If the push fails, report the error and stop — do not force-push.

If the branch is already in sync with the remote, proceed to Step 3.

---

## Step 3 — Gather context for the PR

### Branch name and Linear issue ID

Parse the branch name for a Linear issue ID. Common patterns:
- `feature/OF-1234-some-slug`
- `fix/WW-456-description`
- `OF-1234-slug`

Extract the issue ID (e.g. `OF-1234`, `WW-456`) if present. You will need it for the PR title.

### Commit log

Fetch the commits on this branch that are not on the base branch:
```bash
git log --oneline $(git merge-base HEAD origin/<base>)..HEAD
```

Use the full commit messages (not just `--oneline`) to understand the work done:
```bash
git log --format="%s%n%b" $(git merge-base HEAD origin/<base>)..HEAD
```

### Diff summary

Get a high-level sense of the files changed:
```bash
git diff --stat $(git merge-base HEAD origin/<base>)..HEAD
```

---

## Step 4 — Determine the target repo

Try to detect the GitHub remote automatically:
```bash
git remote get-url origin
```

Parse the owner and repo from the URL (handles both `https://github.com/owner/repo.git` and `git@github.com:owner/repo.git`).

If parsing fails, ask the user: **"Which repo?"**

---

## Step 5 — Compose the PR title and description

### Title

Format: `[ISSUE-ID] Contextual title derived from the work`

- If a Linear issue ID was found, prefix the title with it in brackets: `[OF-1234]`
- Derive the title from the commit messages and diff — make it specific and human-readable, not just the branch name slug
- Keep it under 72 characters
- Use sentence case

Examples:
- `[OF-1234] Add bulk CSV export for investor reporting`
- `[WW-89] Fix token expiry not refreshing on silent auth`

### Description

Write a description aimed at an engineer reviewer. Structure it as follows:

```
## What

[1–3 sentence summary of what this PR does and why. Include the motivation or ticket context if inferable from commits.]

## How

[Bullet list of the key implementation decisions — what was changed, added, or removed and the reasoning. Be specific: name files, functions, or APIs touched where helpful.]

## How to test

[Step-by-step instructions a reviewer can follow to verify the changes work. Include:
- Setup steps if any (migrations, env vars, seed data)
- The specific flows to exercise (happy path and at least one edge case)
- What the expected outcome looks like]

## Concerns / notes

[Any risks, trade-offs, known limitations, or things the reviewer should pay special attention to. If there are none, omit this section entirely.]
```

Populate each section from the commit messages, diff, and any context available. Do not leave placeholder text — if a section has nothing meaningful to say, omit it rather than filling it with filler.

---

## Step 6 — Create the PR

Use the base branch resolved in Step 0.

Create it with `gh`. This works in any harness that has the GitHub CLI authenticated:

```bash
gh pr create \
  --title "<composed title>" \
  --body "<composed description>" \
  --base <base> \
  --head <current-branch>
```

If a GitHub MCP server is connected, its `create_pull_request` tool is equivalent — same fields (`owner`, `repo`, `title`, `body`, `head`, `base`). Call the tool this session actually exposes. Claude Code prefixes it `mcp__plugin_github_github__create_pull_request`; Cursor and others do not. Do not call a prefixed name unless that exact tool is in this session. If the MCP call fails, use `gh`.

---

## Step 7 — Report success

Print the full PR URL to the user:

> "Pull request created: https://github.com/<owner>/<repo>/pull/<number>"

If the MCP or CLI returned a URL, use that directly. If only a PR number was returned, construct the URL from the known owner/repo/number.

---
name: raise-pr
description: Raise a GitHub pull request from the current branch — checks for uncommitted changes and a CHANGELOG entry, syncs with remote, then creates a PR against the repo's default branch with a plain-English TL;DR and an engineer-focused description. Run only when the user explicitly asks to raise a PR.
---

# Raise a Pull Request

`/raise-pr` opens a PR from the current branch against the repo's default branch.

---

## Step 1 — Resolve the base branch

```bash
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||'
```

If that's empty, try `git remote show origin | sed -n 's/.*HEAD branch: //p'`. If it's still empty, ask. Don't guess. Call the result `<base>` below. The personal repos all use `main`, but don't hardcode it.

## Step 2 — Pre-flight

- **Working tree** (`git status --short`): if anything is **staged** but not committed, stop and ask the user to commit or stash it. If there are only unstaged or untracked files, list them and ask whether to continue without them.
- **Branch:** if it's `main` or `master`, confirm before continuing.
- **Changelog:** if a root `CHANGELOG.md` exists and `git diff --stat $(git merge-base HEAD origin/<base>)..HEAD -- CHANGELOG.md` is empty, tell the user the branch has no changelog entry (`/commit` writes one) and ask whether to go ahead without it. Don't write the entry yourself. If there is an entry, check that it sits under `## [Unreleased]`, names the Linear ID, and opens with a plain-English sentence. Mention any gaps without blocking.
- **Push:** if `git log --oneline @{u}..HEAD` shows unpushed commits, or there's no upstream, ask, then run `git push -u origin HEAD`. If the push fails, report it and stop. Never force-push.

## Step 3 — Gather context

- The Linear ID from the branch name (`feature/OF-1234-slug` → `OF-1234`).
- Full commit messages: `git log --format="%s%n%b" $(git merge-base HEAD origin/<base>)..HEAD`
- Changed files: `git diff --stat $(git merge-base HEAD origin/<base>)..HEAD`
- Owner/repo from `git remote get-url origin`.

## Step 4 — Write the title and description

**Title:** `[OF-1234] Specific, sentence-case summary`, under 72 characters, derived from the work rather than the branch slug. Leave off the bracket prefix if there's no Linear ID.

**Description:**

```markdown
## TL;DR
<Two or three plain-English sentences a non-engineer could follow: what this changes for users,
and anything the reviewer needs to do or decide. No file names or jargon.>

## What & why
<The change and its motivation, including ticket context.>

## How
<Key implementation decisions: files, functions, APIs touched, and the reasoning.>

## How to test
<Setup (migrations, env vars, seed data), the flows to exercise including one edge case, and the expected result.>

## Concerns
<Risks, trade-offs, known gaps. Omit if none.>
```

Omit any section with nothing real to say. Don't leave placeholders.

## Step 5 — Create it

```bash
gh pr create --title "<title>" --body "<description>" --base <base> --head <branch>
```

A connected GitHub MCP server's `create_pull_request` (`owner`, `repo`, `title`, `body`, `head`, `base`) works too. Call it by whatever name this session exposes, and fall back to `gh` if it fails.

## Step 6 — Report

Give one plain sentence and the link, e.g. "PR is up for review: https://github.com/<owner>/<repo>/pull/<n>". Add anything the user should know, such as a missing changelog entry or unpushed files that were left out.

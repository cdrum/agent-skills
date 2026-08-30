---
name: submit-pr-review
description: Submit the review you just produced to a GitHub PR — posts every item as one review with inline-anchored comments, picks a verdict GitHub will actually accept, then restores the local checkout and removes the review worktree.
---

# Submit a PR Review

The submission half of the review workflow. It takes the review feedback already produced in this conversation — normally by `/review-pr` or `/review-pr-wt` — and:

1. Posts **all** of it, blocking and recommended alike, as a **single** GitHub review.
2. Anchors each item to a line of code where it can; everything else goes in the review body.
3. Submits a verdict, defaulting to **Request Changes** (see Step 3 for the self-authored case).
4. Cleans up: discards review-time local changes, returns to the default branch, removes the review worktree if one exists.

Every step allows an override — the user has final say on the verdict, on which items go, and on how each is worded.

## Usage

`/submit-pr-review [<github-pr-url>]`

Run it right after a review in the same session; it uses the feedback you just produced. With a URL it targets that PR, otherwise it resolves the PR from the review already in the conversation.

---

## Step 1 — Gather the feedback to submit

Collect the review already produced in this conversation. Each item needs:

- A **severity** — blocking (🔴) or recommended (🟡 / nit).
- A **file path and line** if it ties to specific code, so it can be posted inline.
- A **body** — problem, why it matters, suggested fix.
- Plus one **summary paragraph** for the review body.

If there is **no prior review in the conversation**, stop and tell the user to run `/review-pr` or `/review-pr-wt` first, or to paste the feedback they want submitted. Do not invent findings here — this skill submits, it does not review.

---

## Step 2 — Resolve the PR, the author, and the head commit

If a URL was given, parse `owner` / `repo` / `pullNumber` from it; otherwise take them from the review context.

**Always derive `owner/repo` from the git remote — never from a remembered mapping.** The personal repos do not share an owner (`Wendways/wendways`, `OtterFin-ai/otterfin`, `OtterFin-ai/otterfin-cloud`), and a guessed owner 404s:

```bash
git -C <repo-or-worktree> remote get-url origin
```

If a URL was supplied, prefer its owner/repo but sanity-check against the remote.

Then fetch the PR and the authenticated user:

```
mcp__plugin_github_github__pull_request_read { "method": "get", "owner": "<owner>", "repo": "<repo>", "pullNumber": <number> }
mcp__plugin_github_github__get_me {}
```

Note two things:

- The **head commit SHA** (`head.sha`). Inline comments anchor to it, targeting the new file (`side: RIGHT`).
- Whether **the PR author is the authenticated user**. This is the common case in these repos — most Wendways and OtterFin PRs are your own — and it changes what GitHub will accept in Step 3.

---

## Step 3 — Confirm the verdict and the items (override checkpoint)

Map the verdict to the GitHub review `event`:

| Verdict | `event` |
|---------|---------|
| Request Changes (default) | `REQUEST_CHANGES` |
| Approve with suggestions | `COMMENT` |
| Approve | `APPROVE` |

**If the PR author is the authenticated user, the only accepted event is `COMMENT`.** GitHub rejects `APPROVE` and `REQUEST_CHANGES` on your own pull request with a 422 ("Can not approve your own pull request"). Do not attempt them and then report the failure as a problem — silently default to `COMMENT` and say so:

> "This is your own PR, so GitHub only accepts a plain comment review — posting the 6 items as comments rather than Request Changes. The blocking items are still marked 🔴 in the body."

Then present the plan before posting anything: the verdict, and a numbered list of every item that will go, each tagged inline or general with its severity. Honor any override — change the verdict, drop / add / merge / reword an item, re-tag blocking ↔ recommended, re-anchor a line, move an item inline ↔ general.

Do not proceed until the user confirms.

---

## Step 4 — Find exact line numbers for inline items

For each item tied to code, get the line as it exists in **the PR's version of the file at the head SHA**, not in your locally edited copy:

```bash
grep -n "<distinctive snippet>" <path>
```

For a multi-line range, note the first and last line (`startLine` … `line`).

If an item cannot be tied cleanly to a line — an architectural concern, a missing test, anything cross-cutting — keep it for the review body rather than forcing a bad anchor.

---

## Step 5 — Create a pending review, attach comments, submit

Post everything as **one** review so the author gets a single notification.

**5a. Open a pending review** (omitting `event` keeps it pending):

```
mcp__plugin_github_github__pull_request_review_write {
  "method": "create", "owner": "<owner>", "repo": "<repo>",
  "pullNumber": <number>, "commitID": "<head-sha>"
}
```

**5b. Add each inline item:**

```
mcp__plugin_github_github__add_comment_to_pending_review {
  "owner": "<owner>", "repo": "<repo>", "pullNumber": <number>,
  "path": "<file path>", "line": <line>, "side": "RIGHT",
  "startLine": <first line>, "startSide": "RIGHT",
  "subjectType": "LINE",
  "body": "<severity marker + problem + why + fix>"
}
```

Post **every** agreed item, nits included. Prefix each with its severity (🔴 blocking, 🟡 recommended, `Nit:`) so it can be triaged at a glance.

**5c. Submit** with the confirmed event and a summary body — what the PR does, what you verified, then the headline issues:

```
mcp__plugin_github_github__pull_request_review_write {
  "method": "submit_pending", "owner": "<owner>", "repo": "<repo>",
  "pullNumber": <number>, "event": "<REQUEST_CHANGES|COMMENT|APPROVE>",
  "body": "<summary>"
}
```

- If `add_comment_to_pending_review` reports no pending review, re-run 5a.
- If an inline comment is rejected because the line is not in the diff, move that item to the review body — do not retry a bad anchor.
- `gh` is the fallback if the MCP is unavailable, but the GraphQL path for review comments is fiddly; prefer the MCP tools.

---

## Step 6 — Clean up the local environment

Return the checkout to a clean state. Confirm before discarding anything that might be the user's own work rather than a review byproduct.

**6a. Discard review-time changes** — scratch files, applied suggestion diffs, a migration you generated while checking one:

```bash
git -C <repo-or-worktree> status --short   # look first
git -C <repo-or-worktree> restore .        # discard tracked edits
git -C <repo-or-worktree> clean -nd        # DRY RUN — read this before the next line
git -C <repo-or-worktree> clean -fd        # remove untracked byproducts
```

`git clean -fd` is destructive and these repos keep untracked local files that matter (`.env`, `.env.local`, `backups/`, seed dumps). Never run it without reading the `-nd` output first, and never run it in the main checkout unless the user confirms each path.

**6b. Remove the worktree**, if `/review-pr-wt` created one:

```bash
git worktree remove ../<repo>-pr-<number>
git branch -D pr-<number>
```

If it refuses because of remaining changes, report that and let the user choose between `--force` and keeping it.

**6c. Return to the default branch** — only relevant for the non-worktree `/review-pr` flow, where the PR branch was checked out over the user's tree. Resolve the branch instead of assuming it:

```bash
git symbolic-ref --short refs/remotes/origin/HEAD | sed 's|^origin/||'   # `main` in every personal repo
git checkout <default-branch>
```

If the user had their own branch checked out before the review, return to **that** branch, not the default one — check the conversation for what `/review-pr` recorded before switching.

Then confirm the end state:

```bash
git worktree list
git branch --show-current
git status --short
```

---

## Step 7 — Report

Briefly:

- The PR number, review URL, and the event actually submitted (noting if it was downgraded to `COMMENT` because it is your own PR).
- A short table of posted items — severity · `file:line` or "general".
- Confirmation that the worktree was removed, the branch is back where it started, and the tree is clean.

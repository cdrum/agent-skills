---
name: submit-pr-review
description: Submit the review you just produced to a GitHub PR — posts every item as one review with inline-anchored comments, picks a verdict GitHub will actually accept, then restores the local checkout. Run only when the user explicitly asks to submit.
---

# Submit a PR Review

Posts the review already produced in this conversation (normally by `/review-pr`) as **one** GitHub review, with line-anchored comments where possible, then cleans up. This skill submits reviews; it doesn't write them. If there's no review in the conversation, ask the user to run `/review-pr` first or paste the feedback. Don't invent findings.

`/submit-pr-review [<github-pr-url>]`

---

## Step 1 — Collect the items

From the review, take the TL;DR and every item with its tag (🔴 must fix, 🟡 should fix, Nit) and its `path:line` anchor, or "general" if it has none.

## Step 2 — Resolve the PR and author

Take owner/repo from the git remote (`git -C <repo-or-worktree> remote get-url origin`), never from memory. If a URL was given, prefer it but sanity-check it against the remote. Then:

```bash
gh pr view <number> --repo <owner>/<repo> --json author,headRefOid,url
gh api user --jq .login
```

Note the head SHA (inline comments anchor to it) and whether the PR author is the authenticated user.

## Step 3 — Confirm the verdict and items

| Verdict | `event` |
|---|---|
| Request changes (default if there's any 🔴) | `REQUEST_CHANGES` |
| Approve with suggestions | `COMMENT` |
| Approve | `APPROVE` |

**On your own PR, GitHub accepts only `COMMENT`** (anything else returns a 422). Use `COMMENT` and say so in one line. This is the common case in these repos.

Show the plan: the verdict and a numbered list of every item with its tag, marked inline or general. Apply any changes the user asks for (reword, drop, merge, re-tag, re-anchor), and don't post until they confirm.

## Step 4 — Check line numbers

Anchor each inline item to its line **in the PR's version of the file at the head SHA**, not in any locally edited copy (`grep -n "<snippet>" <path>`). Put cross-cutting items (architecture, missing tests) in the body rather than forcing a bad anchor.

## Step 5 — Post one review

Post every agreed item, nits included, each starting with its tag. The body leads with a plain-English TL;DR, meaning the verdict and what the author needs to do, before any technical detail:

```bash
gh api --method POST "repos/<owner>/<repo>/pulls/<number>/reviews" --input - <<'EOF'
{
  "commit_id": "<head-sha>",
  "event": "<REQUEST_CHANGES|COMMENT|APPROVE>",
  "body": "**TL;DR:** <plain-English verdict and what to do>\n\n<general items and notes>",
  "comments": [
    { "path": "<file>", "line": <line>, "side": "RIGHT", "body": "🔴 <problem> — <why> — <fix>" }
  ]
}
EOF
```

For a multi-line range, add `"start_line"` and `"start_side": "RIGHT"`. If GitHub rejects a comment because its line isn't in the diff, move that item into the body and post again.

A connected GitHub MCP server works too: `pull_request_review_write` (create pending), `add_comment_to_pending_review` per item, then `pull_request_review_write` with `submit_pending`. Call these by whatever names this session exposes, and fall back to `gh` if they fail.

If the user wants follow-up Linear issues created from any item, each issue description opens with a plain-English **TL;DR** (what's wrong and why it matters to users), then the technical detail and a link to the review.

## Step 6 — Clean up

Confirm before discarding anything that might be the user's own work rather than something the review created.

- **Review byproducts** (scratch files, applied suggestions): check `git status --short` first, then `git restore .`. For untracked files, run `git clean -nd` and read the output before running `git clean -fd`. These repos keep untracked files that matter (`.env*`, `backups/`, seed dumps), so in the main checkout, confirm each path with the user.
- **Worktree review:** `git worktree remove ../<repo>-pr-<number>` then `git branch -D pr-<number>`. If removal refuses because of leftover changes, let the user choose between `--force` and keeping it.
- **In-place review:** ask whether to stay on the PR branch (for local testing) or return to the branch `/review-pr` recorded before checkout.

## Step 7 — Report

Start with one plain sentence, for example: "Posted your review on PR #42 as Request changes, with 5 comments. Your checkout is back on `feature/of-91-…`." Then give the review URL, the event actually sent (and whether it was switched to `COMMENT`), and a short table of the items posted (tag · `path:line` or general).

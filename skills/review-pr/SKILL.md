---
name: review-pr
description: Review a GitHub pull request — checks the branch out in the current working tree (or in a separate git worktree only when explicitly asked), then gives an opinionated review covering security, tests, performance, migrations, and project conventions.
---

# Review a Pull Request

## Usage

`/review-pr [<github-pr-url> | <repo> <pr-number> | <pr-number>] [in a worktree]`

- `/review-pr 81` uses the current repo
- `/review-pr wendways 81` or a full PR URL
- `/review-pr` with no argument asks for the repo and lists open PRs
- Add "in a worktree" (or "wt") to review without touching the current checkout

**Default: check the branch out in the current working tree** so it can be run and tested locally. Use a worktree **only** when the user explicitly asks for one.

---

## Step 1 — Resolve the PR

Read the owner from the remote, never from memory. Wendways lives under `Wendways` and OtterFin under `OtterFin-ai`, so a guessed owner 404s:

```bash
git remote get-url origin
```

- **URL given:** parse owner/repo/number from it; sanity-check the owner against the remote.
- **Repo + number:** combine with the remote's owner. A value containing `/` is already a full `owner/repo` slug.
- **Number only:** use the current repo.
- **Nothing:** ask which repo, then list open PRs with `gh pr list --repo <owner>/<repo> --state open --json number,title,headRefName,author` and ask which one.
- **Not in a git repo:** ask for the full `owner/repo`.

Use `gh` for GitHub calls. A connected GitHub MCP server's tools (`list_pull_requests`, `pull_request_read`) work too; call them by whatever name this session exposes. If neither is available, say so and stop.

## Step 2 — Fetch PR details

```bash
gh pr view <number> --repo <owner>/<repo> --json title,body,author,headRefName,baseRefName,files,url
```

## Step 3 — Get the code

Record the current branch first (`git branch --show-current`) so `/submit-pr-review` can return to it later.

### Default: check out in place

If `git status --short` shows uncommitted changes, tell the user and ask whether to continue. Then:

```bash
gh pr checkout <number> --repo <owner>/<repo>
```

If checkout fails, report the error and stop. Don't force anything.

### Only when asked: separate worktree

Build a worktree from the PR's head ref so the current checkout is untouched. Don't use `gh pr checkout` here, because it switches the current tree's branch:

```bash
git fetch origin pull/<number>/head:pr-<number>   # add --force if pr-<number> already exists
git worktree add ../<repo>-pr-<number> pr-<number>
```

Check that `git branch --show-current` still returns the recorded branch. If it doesn't, stop and tell the user. Run every later git or file command against the worktree path (`git -C ../<repo>-pr-<number> …`).

A fresh worktree has no `node_modules`, no database, and (for `otterfin-cloud`) an empty `otterfin` submodule until you run `git -C <path> submodule update --init --recursive`. The review reads code, so installing is rarely needed. If a finding needs the test suite, say so.

## Step 4 — Load the project's rules

Read `AGENTS.md` / `CLAUDE.md` at the repo root. Where it disagrees with this skill, it wins. These are the project rules that matter most in review:

**OtterFin** (`otterfin`, `otterfin-cloud` — pnpm/turbo monorepo: `apps/community`, `packages/{auth,core,db,ui}`, `plugins/`)
- Every tenant query is scoped by `household_id` taken from the session, never from client input. A lookup by ID alone is a blocking IDOR. `tests/isolation/` must pass.
- Money is integer cents. Flag float math on amounts.
- No `@supabase/supabase-js` or `@vercel/*` in app code. DB via Prisma, auth via Auth.js.
- Zod on every external input. shadcn/ui + Tailwind only, with `dark:` variants from the start.
- No PII in logs (amounts, descriptions, account names, emails). Log only IDs, action, and timestamp.
- The OSS repo holds no billing, plans, trials, or premium flags. First-party files carry the AGPLv3 header.
- `otterfin-cloud` is the private premium layer over the `otterfin` submodule (Vercel + Supabase, versions `X.Y.Z-cloud.N`).

**Wendways** (single Next.js app — `src/{app,server,core,db,ui}`, tRPC + Better Auth + Capacitor)
- Every user-owned query is scoped by `user_id` from the session. A lookup by ID alone is a blocking IDOR. `src/tests/isolation/` must pass.
- `src/core` stays framework-agnostic: no React or Next imports.
- No `@vercel/*`, Render-specific, or Supabase data clients. Storage and email go through their adapters.
- Reference data is keyed by the source's stable ID, never IATA. Nothing is filtered out; removed rows are marked retired, never deleted.
- Distance is great-circle per segment in `src/core`. Missing coordinates mean blank, never a guess.
- Times are stored in UTC plus an IANA zone. Flag anything that assumes the server's timezone.
- No PII in logs (trips, places, locations, emails). No license headers.

**Both:** `prisma db push` is forbidden. Never edit an already-committed migration; add a new one. If a PR edits an existing migration and its linked issue doesn't explicitly authorize that, it's blocking. Ask the user if unsure.

## Step 5 — Review

Read the diff (`gh pr diff <number> --repo <owner>/<repo>`) and the surrounding code. Cover security, correctness, tests, performance, migrations, conventions, and PR hygiene (description explains *why*, changelog entry for user-visible changes, no leftover TODOs). Flag real problems, not hypotheticals.

## Step 6 — Write the review

Write for a reader who has to follow the reasoning, not just a list of code findings. An accurate review that mixes terms and shows raw numbers is still hard to act on.

**Rules:**

1. **Define terms once, with one running example.** Right after the TL;DR, add a short **Terms** section. It defines each domain word the findings will use, illustrated with one concrete example taken from the PR. For a Wendways expenses PR, the example might be a booking with a stated total (a HK$10,960 Cathay Pacific booking) containing components (Flight 1, Flight 2), plus a separate standalone component (a hotel). The section then defines "price", "pricing mode", "expense", "split", "trip cost", and any new concept the PR introduces. Skip the Terms section only when the PR has no domain concepts, for example a dependency bump or a lint fix.
2. **Stick to those terms and that example.** Never switch synonyms partway through ("booking" vs. "reservation", "component" vs. "leg" vs. "plan item"). Count things explicitly ("1 split", "2 splits totaling HK$21,920"). Show money in real currency (HK$18,295.00), never in the database's minor units (1,829,500).
3. **Give each Must fix and Should fix finding the same shape, in plain English:** **Setup** (the state of the example before), **Steps** (what the user does), **What happens**, **What should happen**, **Why** (the cause in one or two sentences, with code terms only where they're needed), **Fix**. Put file and line references at the end of the finding, not in the explanation.
4. **Give the scope.** Say plainly what the finding does *not* affect (for example, "no one's balance changes") and how often it's likely to happen.

Nits and non-behavioral findings (naming, dead code, a missing test) can skip the Setup/Steps shape. Give them one plain sentence, then the reference.

Use this layout. Omit any section with nothing in it. Don't add empty headings, praise for code that's merely correct, or a closing recap.

```markdown
## TL;DR
<Verdict: Approve / Approve with suggestions / Request changes.> <One or two plain-English
sentences: what this PR does, whether it's safe to merge, and what the author needs to do.
No file names, function names, or jargon here.>

## Terms
<Running example from the PR, then each domain term defined against it.>

## Must fix
### 🔴 <short title>
- **Setup:** <the example's state before>
- **Steps:** <what the user does>
- **What happens:** <…>
- **What should happen:** <…>
- **Why:** <cause, in one or two sentences>
- **Fix:** <…>
- **Scope:** <what it doesn't affect; how rare it is>
- **Ref:** `path/to/file.ts:123`

## Should fix
### 🟡 <short title>
<same shape; Ref may be "general">

## Nits
- Nit: <one plain sentence> · `path:line`

## Notes
<Anything else a reviewer needs: approach concerns, what you verified, tests you couldn't run.>
```

Keep the 🔴 / 🟡 / Nit tags and the `path:line` in each **Ref** line, because `/submit-pr-review` reads them.

## Step 7 — Prior feedback (re-reviews only)

Skip this unless the PR has an earlier "changes requested" review or unresolved threads (`gh pr view <number> --repo <owner>/<repo> --json reviews`). If it does, judge each prior item on whether the concern is actually fixed, not just whether nearby code changed. Show a checklist:

```
Prior feedback:
  1. [addressed]     "Scope the lookup by householdId" — lib/actions/transaction.ts:212
  2. [not addressed] "Add a test for the expired-token path" — no new test found
```

If every item is addressed, offer to resolve those threads on GitHub. Resolving is GraphQL-only:

```bash
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){pullRequest(number:$n){reviewThreads(first:100){nodes{id isResolved comments(first:1){nodes{body path line}}}}}}}' -F o=<owner> -F r=<repo> -F n=<number>
gh api graphql -f query='mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}' -F id=<thread-id>
```

Resolve only threads the user confirms. List any unaddressed items under "Must fix".

## Step 8 — Leave the environment

If `/submit-pr-review` comes next, leave everything as it is, because that skill handles cleanup. Otherwise:

- **In place:** leave the PR branch checked out (the user usually wants to test it) and remind them which branch they were on before.
- **Worktree:** offer to remove it (`git worktree remove ../<repo>-pr-<number>` then `git branch -D pr-<number>`). If removal refuses because of leftover changes, show them and let the user choose between `--force` and keeping it.

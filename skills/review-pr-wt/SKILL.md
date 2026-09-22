---
name: review-pr-wt
description: Review a GitHub pull request inside a dedicated git worktree — leaves your current branch and uncommitted work untouched, performs an opinionated review covering security, tests, performance, and conventions, then cleans the worktree up.
---

# Review a Pull Request (in a Worktree)

The worktree variant of `/review-pr`. It reviews the PR in a **dedicated git worktree** instead of checking the branch out over your current working tree, so in-progress work is never disturbed. Use it when you have a branch mid-flight (the normal state in these repos) or want to review several PRs side by side.

> Steps 4-6 are deliberately identical to `/review-pr`. If you change the review criteria in one file, change them in the other.

## Usage

`/review-pr-wt [<github-pr-url> | <repo> <pr-number> | <pr-number>]`

Examples:
- `/review-pr-wt https://github.com/OtterFin-ai/otterfin/pull/27`
- `/review-pr-wt wendways 81` (repo name + PR number)
- `/review-pr-wt 81` (PR number only — uses the current repo)
- `/review-pr-wt` (interactive — prompts for repo then shows open PRs)

---

## Step 1 — Resolve the PR

### Resolve the owner from the remote — always, first

**Never hardcode the GitHub owner.** The personal repos do not share one: Wendways lives under `Wendways`, everything OtterFin lives under `OtterFin-ai`. A guessed owner produces a confusing 404 on the PR lookup, so read it from the repo you are standing in:

```bash
git remote get-url origin
```

Parse the owner from the result — `git@github.com:Wendways/wendways.git` → `Wendways`, `https://github.com/OtterFin-ai/otterfin.git` → `OtterFin-ai`. Combine it with the repo name for the full slug.

If the current directory is not a git repo, ask the user for the full `owner/repo` slug.

### If a URL was provided

Parse the owner, repo, and PR number from the URL and skip to Step 2. Prefer the owner from the URL, but sanity-check it against the remote.

### If a repo name and PR number were provided

Resolve the owner as above, combine it with the repo name, and skip to Step 2. A value already containing a `/` is a literal `owner/repo` slug — use it as-is.

### If only a PR number was provided

Resolve both owner and repo from the current directory's remote, and skip to Step 2.

### If nothing was provided

Ask the user: **"Which repo? (otterfin / otterfin-cloud / wendways / …)"**, resolve the owner as above, then list open pull requests:

```bash
gh pr list --repo <owner>/<repo> --state open --json number,title,headRefName,author
```

A connected GitHub MCP server's `list_pull_requests` tool (`owner`, `repo`, `state: "open"`) returns the same list. Call it by the name this session exposes — Claude Code prefixes it `mcp__plugin_github_github__list_pull_requests`; do not use that prefix unless that exact tool is present. If neither `gh` nor the MCP server is available, say so and stop.

Display a numbered list:

```
Open PRs in OtterFin-ai/otterfin:
  1. #42 — Fix auth token expiry (fix/of-123-auth-token-expiry) — opened by alice
  2. #38 — Add bulk export endpoint (feature/of-119-bulk-export) — opened by bob
```

Ask: **"Which PR? (enter a number)"** and resolve the selection to an owner/repo/number.

---

## Step 2 — Fetch PR details

```bash
gh pr view <number> --repo <owner>/<repo> --json title,body,author,headRefName,baseRefName,files,url
```

The GitHub MCP tool `pull_request_read` (`owner`, `repo`, `pullNumber`) is equivalent, under whatever name this session gives it.

Note the head branch name, base branch, title, body, author, and file count.

---

## Step 3 — Create an isolated worktree for the PR

This skill does **not** check the branch out in the current working tree. It creates a dedicated worktree instead, leaving your current branch and any uncommitted changes untouched.

Record the current branch first, so the guarantee can be verified at the end of this step:

```bash
git branch --show-current    # remember this value
```

Fetch the PR's head ref into a local branch, then build the worktree from it. Neither command touches the current working tree:

```bash
git fetch origin pull/<number>/head:pr-<number>
git worktree add ../<repo>-pr-<number> pr-<number>
```

> **Never use `gh pr checkout` here.** It has no worktree-aware flag and acts on the current working directory — it would switch *your* branch, which is exactly what this skill exists to prevent.

Confirm the isolation held before continuing:

```bash
git branch --show-current                            # must equal the value recorded above
git -C ../<repo>-pr-<number> branch --show-current   # must be pr-<number>
```

If the first command returns anything else, **stop and tell the user immediately** — the working tree was disturbed and must be restored before the review continues.

If the local `pr-<number>` branch already exists from an earlier review, either refresh it (`git fetch origin pull/<number>/head:pr-<number> --force`) or pick a fresh name. If the worktree path already exists, `git worktree add` fails — tell the user and ask whether to reuse it or remove it first (`git worktree remove ../<repo>-pr-<number>`).

**Every subsequent git and file operation runs against the worktree path**, never the main checkout. Do not rely on the shell's working directory — pass the path explicitly: `git -C ../<repo>-pr-<number> <command>`, and full worktree-relative paths when reading files.

### What a fresh worktree does not have

- **No `node_modules`.** These are pnpm workspaces, so a new worktree cannot run `pnpm typecheck`, `pnpm lint`, or any test until `pnpm install` runs there — several minutes and a few hundred MB. This skill reviews by reading the diff and the surrounding code, so that is usually unnecessary. If a finding genuinely needs the suite, say so and run it from the main checkout on that branch instead.
- **No database or containers.** `docker compose` services and `DATABASE_URL` point at the original checkout; nothing DB-backed runs in the worktree.
- **No submodule contents.** `otterfin-cloud` carries the `otterfin` community repo as a submodule, and a new worktree leaves it empty. If the PR touches anything spanning that boundary, populate it first:

  ```bash
  git -C ../otterfin-cloud-pr-<number> submodule update --init --recursive
  ```

## Step 4 — Detect the tech stack

Determine the stack from the repo name and the worktree contents:

| Repo | Primary stack |
|------|--------------|
| `otterfin` | Next.js 16 App Router · React 19 · TypeScript (strict) · Tailwind + shadcn/ui · Prisma + PostgreSQL · Auth.js (NextAuth v5) · Zod · Vitest + Playwright · pnpm/turbo monorepo |
| `otterfin-cloud` | Same stack, private premium layer over the `otterfin` submodule · Vercel + Supabase · versions as `X.Y.Z-cloud.N` |
| `wendways` | Next.js 16 App Router · React 19 · TypeScript (strict) · Tailwind + shadcn/ui · tRPC 11 + TanStack Query · Prisma + PostgreSQL · Better Auth · MapLibre + ECharts · Vitest + Playwright · Capacitor (iOS/Android) · single app, `src/` not a monorepo |

**OtterFin specifics** (a pnpm + turbo monorepo — `apps/community`, `packages/{auth,core,db,ui}`, `plugins/`):

- **Multi-tenancy is the headline concern.** Every tenant table has `household_id`; every query must scope by it explicitly and derive the household from the authenticated session, never from client input. Treat a missing `householdId` scope (resource lookup by ID alone → IDOR) as a blocking issue. Isolation tests live in `tests/isolation/` and **must always pass**.
- **Money** is stored as integers in the smallest currency unit (cents) — flag any float arithmetic on amounts.
- **No vendor coupling** — `@supabase/supabase-js` and `@vercel/*` must not appear in application code. All DB access via Prisma, all auth via Auth.js.
- **Migrations** — `prisma db push` is forbidden (causes schema drift); changes go through `prisma migrate dev` / `make db-migrate`. See the Database Migrations section below.
- **No PII in logs** — never log amounts, descriptions, account names, or emails; only `household_id`, `user_id`, action, timestamp.
- **Zod** validation required on every external input (API routes, Server Actions, forms, imports).
- **UI** — shadcn/ui + Tailwind only (no CSS-in-JS, no alternative UI libs); every component needs `dark:` variants from the start.
- **Entitlements/billing** — this OSS repo must not contain billing, plan definitions, trial logic, or premium feature flags.

**Wendways specifics** (single Next.js app — `src/{app,server,core,db,ui}`):

- **Multi-tenancy is per `userId`.** Every user-owned table has `user_id` and every query must include it, derived from the session. A lookup by id alone is a blocking IDOR. Isolation tests live in `src/tests/isolation/` and **must always pass**.
- **`src/core` must stay framework-agnostic** — zero React/Next imports, so the future package extraction stays a folder move. Flag any React/Next import that lands there.
- **No vendor coupling** — no `@vercel/*`, no Render-specific or Supabase data clients. DB via Prisma and a standard `DATABASE_URL`; storage via the S3-compatible adapter; email via the email adapter.
- **Migrations** — `prisma db push` is forbidden; use `prisma migrate dev` and review the generated file in the PR.
- **Reference data** (airports, airlines, cities) is keyed by the source's stable id, **never** by IATA code, and nothing is filtered out (closed airports, defunct airlines, `scheduled_service = no` all stay). Removed entries are marked retired, never hard-deleted.
- **Distance** is great-circle, computed per segment in `src/core`, user-overridable; missing coordinates mean blank, never a guess.
- **Times** are stored UTC plus the relevant IANA timezone — flag anything that assumes the server timezone.
- **No PII in logs** — trip details, place names, locations, and emails are all sensitive; log `user_id`, action, timestamp only.
- **No copyright/license headers** (closed-source commercial), unlike OtterFin which requires an AGPLv3 header on first-party files.

Also read `CLAUDE.md` (and `AGENTS.md`) at the root of the worktree — they contain the authoritative tech stack, architecture rules, security rules, and conventions that override the defaults in this skill.

---

## Step 5 — Review the PR

Read the changed files from `gh pr diff <number> --repo <owner>/<repo>` (or the GitHub MCP) and from the worktree copy as needed. Then produce a structured review under the following sections. Be direct and opinionated — flag real problems, not hypotheticals.

**Only Summary and Verdict are required. Omit every section with no findings** — do not emit a heading followed by "no issues found", "N/A", or a restatement of what you checked. The sections below are a checklist for *you*, not a template for the output; a small PR should routinely produce a review with two or three headings.

Write each finding as: what is wrong · where (`file:line`) · why it matters · the fix. Skip preamble, skip praise for code that is merely correct, and skip a closing paragraph that summarizes the findings you just listed.

---

### Summary

One short paragraph: what the PR does and whether the overall approach makes sense.

---

### Security

Flag any of the following if present:

**Secrets & code execution**
- Secrets, tokens, or credentials committed to source (everything belongs in env config)
- Unsafe use of `eval`, dynamic `import()` of user input, or other dynamic code execution
- Hard-coded secrets or API keys, especially anything reaching client-side code

**Authentication & authorization**
- Missing or bypassable authentication / authorization checks
- Logic that processes a request before verifying auth
- Insecure direct object references (IDOR) — looking up a resource by ID alone without scoping to the authenticated user/owner
- Trusting client-supplied identity or tenant scope instead of deriving it from the session
- In a multi-tenant codebase, any query that omits the tenant scope (treat as blocking)

**Input validation & requests**
- External input (API routes, form handlers, imports, query params) not validated/sanitized
- State-mutating endpoints without CSRF protection
- Unvalidated redirects or open redirects
- User-controlled data used in file paths, shell commands, or SQL without sanitization

**Data handling**
- Raw SQL with string interpolation instead of parameterized queries
- Sensitive data or PII written to logs
- Server-only data leaked to the client (over-broad API responses, server→client prop boundaries)
- Rendering unsanitized content as HTML (XSS, e.g. `dangerouslySetInnerHTML`)

---

### Tests

- Are tests included for new or changed behavior?
- Do the tests cover the happy path and meaningful edge cases?
- Are there any tests that appear to pass trivially without actually verifying behavior?
- Is the right level of test used — unit for pure logic, integration for endpoints/DB access, E2E for critical user flows?
- Are server-side endpoints, DB operations, and UI components/hooks covered with the project's standard test tooling?

If tests are absent for non-trivial logic, call it out clearly.

---

### Performance

**Backend / data access**
- N+1 query patterns (missing eager-loading / batching)
- Unindexed fields used in filters or sorts on large tables
- Expensive operations (file I/O, network calls, heavy computation) running synchronously in a request/response cycle without a task queue
- Queries or serializers fetching more data than the response needs

**Frontend / React**
- Missing `useMemo` / `useCallback` around expensive or frequently re-created values passed as props
- Unnecessary full-page or large-component re-renders
- Large dependencies added without checking bundle impact
- Unoptimized images (e.g. not using the framework's image component)
- Work done on every request that could be cached or moved to build time

---

### Database Migrations

**Never modify an existing migration — always add a new one.** Editing a migration that has already been committed (changing its operations, SQL, or dependencies) rewrites history that other environments may have already applied, and is a defect by default.

If the PR modifies an existing migration file rather than adding a new one, confirm the PR's linked issue **explicitly authorizes** rewriting that migration. If it does not, or you are unsure, **stop and ask the user to confirm** before passing this section. Treat an unauthorized edit to an existing migration as a blocking issue.

---

### Conventions & Code Quality

- Does the code follow patterns already established in this repo?
- Are there violations of the rules in `CLAUDE.md` (if found)?
- Naming: are variables, functions, and components named clearly and consistently?
- Are there any copy-paste blocks that should be extracted?
- Dead code, unused imports, or leftover debug statements?
- Idiomatic use of the language and framework, and adherence to the project's lint/style rules?
- For React/Next.js: consistent component structure, `use client` / `use server` boundaries correct?

---

### Changelog & PR Hygiene

- Does the PR description explain the *why*, not just the *what*?
- Is there a changelog entry or release note if this is a user-facing change?
- Are there any TODO/FIXME comments left that should be resolved before merge?

---

## Step 6 — Verdict

End with one of three verdicts and a brief rationale:

| Verdict | Meaning |
|---------|---------|
| **Approve** | Ready to merge with no required changes |
| **Approve with suggestions** | Safe to merge; suggestions are non-blocking improvements |
| **Request changes** | One or more issues must be addressed before merge |

List any blocking issues clearly under the verdict.

---

## Step 7 — Resolve prior review feedback (re-reviews only)

This applies **only when the PR already has a review that requested changes** (or unresolved review threads) and the author has since pushed updates meant to address them. Detect it:

```bash
gh pr view <number> --repo <owner>/<repo> --json reviews,comments
gh api "repos/<owner>/<repo>/pulls/<number>/comments"
```

The GitHub MCP tool `pull_request_read` with `method` `get_reviews` and `get_review_comments` returns the same data, under whatever name this session gives that tool.

If there is no prior "changes requested" review and no outstanding threads, **skip this step entirely**.

Otherwise judge each prior item on whether the concern is genuinely resolved — not merely that something nearby changed — and show the user a checklist:

```
Prior review feedback:
  1. [addressed]     "Scope the lookup by householdId" — now scoped in lib/actions/transaction.ts:212
  2. [addressed]     "Extract the duplicated transfer mapping" — deriveTransferMetadata, transaction.ts:64
  3. [not addressed] "Add a test for the expired-token path" — no new test found
```

If every applicable item is addressed, ask:

> **"The prior review feedback looks addressed. Mark the resolved review threads as resolved in GitHub? (y/n)"**

On yes, resolve them with `gh` — thread resolution is GraphQL-only:

```bash
# 1. List unresolved threads and their IDs
gh api graphql -f query='
  query($owner:String!, $repo:String!, $pr:Int!) {
    repository(owner:$owner, name:$repo) {
      pullRequest(number:$pr) {
        reviewThreads(first:100) {
          nodes { id isResolved isOutdated comments(first:1){ nodes { body path line } } }
        }
      }
    }
  }' -F owner=<owner> -F repo=<repo> -F pr=<number>

# 2. Resolve each agreed thread by ID
gh api graphql -f query='
  mutation($threadId:ID!) {
    resolveReviewThread(input:{threadId:$threadId}) { thread { id isResolved } }
  }' -F threadId=<thread-node-id>
```

- Resolve only threads the user confirmed. Never resolve one whose feedback is unaddressed.
- If `gh auth status` fails, report it and skip resolution rather than silently trying another path.
- Leave unaddressed items open and call them out under the verdict as blocking.
- `gh` talks to the remote, so it behaves identically from the worktree.

---

## Step 8 — Clean up the worktree

After delivering the verdict, ask: **"Remove the review worktree at `../<repo>-pr-<number>`? (y/n)"**

On yes:

```bash
git worktree remove ../<repo>-pr-<number>
git branch -D pr-<number>   # only if the local pr-<number> branch is no longer needed
```

`git worktree remove` refuses when the worktree has changes — including a `pnpm install` you ran there, since `node_modules` is gitignored but `pnpm-lock.yaml` may have moved. Report what is there and let the user choose between `--force` and keeping it.

If the user wants to keep the worktree (to keep iterating, or to run the suite there), leave it and remind them it can be removed later with `git worktree remove ../<repo>-pr-<number>`.

If `/submit-pr-review` runs next, leave the worktree in place — that skill posts the review and then cleans up.

---

## Reviewing several PRs at once

This skill reviews one PR per invocation. To review several in parallel, use one worktree and one agent session per PR:

```bash
git fetch origin pull/27/head:pr-27
git worktree add ../otterfin-pr-27 pr-27
```

Open `../otterfin-pr-27` in a separate session and run `/review-pr-wt 27` there.

Same rule as Step 3: fetch the head ref into a local branch, then build the worktree from it. Never `gh pr checkout` — it switches the branch of whichever tree you run it in.

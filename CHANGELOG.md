# Changelog

All notable changes to this repo's Claude Code skills are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Changed

- `/review-pr` findings are now written so a reader can follow them without reading the code. Each review defines its domain terms once against one running example from the PR, then keeps to those terms. Every finding follows the same shape: Setup, Steps, What happens, What should happen, Why, Fix, Scope. Money appears in real currency, never database minor units, and file references go at the end. `/submit-pr-review` carries the Terms section and this shape into the posted GitHub review
- Every skill's output now opens with a plain-English TL;DR, so the result and next step come before the technical detail. This covers chat reports, PR descriptions, review bodies, changelog entries, and Linear issues a skill creates
- `/review-pr` now covers both review modes: by default it checks the branch out in your working tree for local testing, and it uses a separate worktree only when you ask for one. `/review-pr-wt` is removed. The review output uses 🔴 / 🟡 / Nit tags and `path:line` anchors, which `/submit-pr-review` reads directly
- Skills are trimmed for current models: generic review checklists, repeated tool-prefix boilerplate, filler steps, and all-caps emphasis are gone, and the project-specific rules and guardrails stay
- `/commit`, `/raise-pr`, and `/submit-pr-review` state that they run only on explicit request. This is written in the skill text rather than in Claude Code-only frontmatter, so the skills still upload to claude.ai
- `CLAUDE.md` now imports `AGENTS.md` with `@AGENTS.md`, so Claude Code loads it automatically. The install steps no longer create `~/.claude/commands/` symlinks, because skills are already slash commands
- Each skill is now a directory with `SKILL.md` (`skills/<name>/SKILL.md`) so the same files can be symlinked into Cursor and Claude Code skill directories
- Skill steps no longer call Claude Code tool ids. MCP steps use the server's own tool name (`get_issue`, `create_pull_request`) and tell the agent to discover how this session prefixes it. GitHub steps also have a `gh` path. `/validate-fe` drives whichever browser tools the session has, and `/humanize` no longer reads `$ARGUMENTS`
- Repo instructions now live in `AGENTS.md`. `CLAUDE.md` is a stub that points there

### Added

- Initial set of skills: `/commit`, `/start-issue`, and `/humanize`
- `/raise-pr` skill to raise a GitHub pull request from the current branch
- `/review-pr` skill to review a GitHub pull request
- `/review-pr-wt` — reviews a PR in a dedicated git worktree instead of checking the branch out over the current working tree, so in-progress work is untouched; includes re-review thread resolution and worktree cleanup. Ported from the Alternatives skills library and rewritten for these repos. Verified end-to-end against a real PR
- `/submit-pr-review` — posts a produced review to GitHub as a single review with inline-anchored comments. Defaults to `COMMENT` when the PR is your own, because GitHub rejects `APPROVE` and `REQUEST_CHANGES` from the author with a 422
- `/validate-fe` — drives Chrome through the Claude in Chrome extension (not Playwright MCP) to walk the routes a branch touched, checking console errors, light and dark rendering, and the changed behavior. Knows both apps' dev ports (Wendways 4020, OtterFin 4010) and route maps
- `/draft-release` — drafts an `oss-website` Updates post and a GitHub release body from an OtterFin version's `CHANGELOG.md` section, commits, and Linear issues. Complements the app repos' own `/release` command rather than duplicating it; refuses to announce `otterfin-cloud` changes on the public site
- README with skill list, installation, and authoring instructions
- CLAUDE.md with repo conventions for adding and editing skills

### Fixed

- `/validate-fe` diffed against a hardcoded `origin/main`. It now looks up the repo's default branch
- `/start-issue` called four Linear MCP tools that no longer exist (`update_issue`, `get_viewer`, `get_workflow_states`, `get_labels`). It now uses `save_issue` with `assignee: "me"` and a status name, which also removes two lookup round-trips
- `/start-issue` branched off a base-branch table inherited from the work repos (`dmp`/`admin` → `dev`, `altpe` → `develop`, defaulting to `dev`). It now resolves the default branch from `origin/HEAD` — `main` in every personal repo — and its label→prefix map handles both `Bug`/`Feature` and namespaced `type:*` labels
- `/raise-pr` hardcoded `dev` as the PR base, contradicting its own description. It now targets the resolved default branch, and its changelog gate checks the root `CHANGELOG.md` instead of the `changelogs/` fragments these repos do not use
- `/review-pr` mapped the `otterfin` shortcode to `otterfin/otterfin`; the real remote is `OtterFin-ai/otterfin`, so every lookup 404'd. Owner is now derived from `git remote get-url origin`, which also covers Wendways living under a different owner
- `/review-pr` gained Wendways and `otterfin-cloud` stack profiles, and now omits review sections that have no findings instead of printing empty headings

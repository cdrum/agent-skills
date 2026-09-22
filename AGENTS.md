# AGENTS.md

This repo contains my personal agent skills, loaded by Claude Code, Cursor, and any other harness that reads `SKILL.md`.

## Conventions

- One skill per directory under `skills/<name>/SKILL.md`.
- Directory name must match the `name` field in frontmatter (kebab-case).
- The `description` frontmatter field must be a single, specific sentence — it drives both the skill picker UI and auto-invocation logic.
- Skill prompts should be self-contained: assume no prior conversation context.
- Prefer imperative phrasing in prompts ("Review the…", "Generate a…").
- Keep frontmatter portable: only `name`, `description`, and the Agent Skills standard fields (`license`, `compatibility`, `metadata`, `allowed-tools`). Claude Code-only keys such as `disable-model-invocation` or `argument-hint` make the claude.ai upload fail, and that upload is how cloud sessions get these skills. If a skill has side effects and should run only on request, say so in its description and body.
- Call MCP tools by the server's own name (`get_issue`, `create_pull_request`) and say to use whatever name the session exposes. Never write a harness-specific prefix like `mcp__…`. Give a `gh` fallback for GitHub.
- Write for a capable model: state the goal, the project-specific rules, and the real guardrails once each. Skip generic checklists the model already knows, filler steps ("analyze the changes"), and ALL-CAPS emphasis.

## Plain English first

Anything a skill produces for a person to read opens with a plain-English **TL;DR**: the result and what, if anything, the reader needs to do, with no jargon, file names, or IDs. Technical detail follows. This applies to:

- the final report a skill gives in chat
- GitHub content: PR descriptions and review bodies
- `CHANGELOG.md` entries (each entry's first sentence is plain English, then the technical detail)
- Linear issues a skill creates, for example follow-ups from a PR review

Commit messages keep the Conventional Commits format.

Each skill states this rule itself, because this file isn't loaded when a skill runs inside another repo.

## When adding or editing a skill

- Verify the skill works end-to-end before committing.
- Update the description if the behavior changes — stale descriptions break auto-invocation.
- Don't add skills that duplicate behavior the harness already provides.

## What not to do

- Don't store secrets, internal URLs, or environment-specific config in skill files — the repo may be shared publicly.
- Don't add skills without testing them.

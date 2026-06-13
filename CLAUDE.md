# Claude Code guidance for this repo

This repo contains my personal Claude Code skills.

## Conventions

- One skill per file under `skills/`.
- Filename must match the `name` field in frontmatter (kebab-case).
- The `description` frontmatter field must be a single, specific sentence — it drives both the skill picker UI and auto-invocation logic.
- Skill prompts should be self-contained: assume no prior conversation context.
- Prefer imperative phrasing in prompts ("Review the…", "Generate a…").

## When adding or editing a skill

- Verify the skill works end-to-end before committing.
- Update the description if the behavior changes — stale descriptions break auto-invocation.
- Don't add skills that duplicate built-in Claude Code behavior.

## What not to do

- Don't store secrets, internal URLs, or environment-specific config in skill files — the repo may be shared publicly.
- Don't add skills without testing them.

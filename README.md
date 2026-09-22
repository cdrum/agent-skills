# Agent Skills

My personal collection of agent skills.

## What are these?

Each skill is a directory containing `SKILL.md`. The directory name matches the `name` field in that file's frontmatter. Claude Code, Cursor, and any other harness that loads this layout can run them. Steps that talk to Linear or GitHub name the MCP server's own tool (`get_issue`, `create_pull_request`) and also give a `gh` command where one exists, so no harness-specific tool prefix is ever required.

Everything a skill produces for a person to read opens with a plain-English TL;DR before the technical detail.

> **Note on the npx approach:** You may have seen repos like [mattpocock/skills](https://github.com/mattpocock/skills) that use `npx skills@latest add ...` to install commands. That's a custom npm tool Matt built — it's not a Claude-native mechanism. The native approach is just copying files into the right directory.

## Skills

| Command | Description |
|---|---|
| `/start-issue` | Start work on a Linear issue — assigns it to you, moves it to In Progress, and creates a correctly prefixed branch off the repo's default branch |
| `/commit` | Commit completed work into Git at a logical milestone, with a conventional commit message, a `CHANGELOG.md` entry, and optional Linear task reference |
| `/validate-fe` | Validate front-end changes in a real Chrome tab — walks the affected routes and checks console errors, light and dark rendering, and the behavior that changed |
| `/raise-pr` | Raise a GitHub PR from the current branch — pre-flight checks, changelog check, push if needed, then a PR against the repo's default branch |
| `/review-pr` | Review a GitHub PR — checks the branch out locally (or in a separate worktree when you say "in a worktree"), then covers security, tests, performance, migrations, and conventions |
| `/submit-pr-review` | Submit the review you just produced to the PR — one review, inline-anchored comments, a verdict GitHub will accept, then restores the local checkout |
| `/draft-release` | Draft public-facing release notes for an OtterFin version — an Updates post for the website plus a GitHub release body |
| `/humanize` | Remove AI writing patterns from a file and rewrite it to sound authentically human |

Skills assume the projects they were written against: Wendways (`Wendways/wendways`) and OtterFin (`OtterFin-ai/otterfin`, `OtterFin-ai/otterfin-cloud`). They read each repo's own `CLAUDE.md` / `AGENTS.md` and treat it as authoritative where the two disagree.

## Installation

First, clone the repo.

### Install all skills

From inside the cloned repo, symlink every skill directory. Cursor reads `~/.cursor/skills/`. Claude Code (CLI and desktop app) reads `~/.claude/skills/`, and each skill there is invocable as `/<name>`, so no `~/.claude/commands/` link is needed.

```bash
for d in "$(pwd)/skills"/*/; do
  name="$(basename "$d")"
  ln -sfn "$d" ~/.cursor/skills/"$name"
  ln -sfn "$d" ~/.claude/skills/"$name"
done
```

### Install a single skill

```bash
ln -sfn "$(pwd)/skills/commit" ~/.cursor/skills/commit
ln -sfn "$(pwd)/skills/commit" ~/.claude/skills/commit
```

Replace `commit` with whichever skill you want. Run these from inside the cloned repo.

### After installing

The skills are available as `/commit`, `/start-issue`, and so on. Cursor picks them up in a new Agent chat. Claude Code picks them up in the next session.

### Cloud sessions

Cloud sessions can't read this machine's `~/.claude/skills`. Skills uploaded to claude.ai sync to Claude Code (locally they show up as `anthropic-skills:<name>`), and that upload is how they reach other environments. Re-upload a skill after changing it here, or the cloud copy goes stale.

### Keeping up to date

```bash
cd ~/Development/agent-skills && git pull
```

Symlinks point at the repo files directly, so pulled changes take effect immediately — no re-linking needed.

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md` — see the frontmatter format below. The directory name and the `name` field must match.
2. Test it locally by symlinking it and running it in a real session.
3. Commit it with a short description of what it does and when to use it.

### Skill file format

```markdown
---
name: skill-name
description: One-line description of what this skill does and when to use it.
---

# Skill title

## Instructions

...
```

Keep the `description` specific, because agents use it to decide whether to invoke the skill automatically. Keep frontmatter to portable fields (see `AGENTS.md`) so the skill still uploads to claude.ai.

---

&copy; 2026 Chris Drumgoole. All rights reserved.

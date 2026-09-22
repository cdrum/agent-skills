# Agent Skills

My personal collection of agent skills.

## What are these?

Each skill is a directory containing `SKILL.md`. The directory name matches the `name` field in that file's frontmatter. Claude Code, Cursor, and any other harness that loads this layout can run them. Steps that talk to Linear or GitHub name the MCP server's own tool (`get_issue`, `create_pull_request`) and also give a `gh` command where one exists, so a harness-specific prefix like `mcp__claude_ai_Linear__…` is never required.

> **Note on the npx approach:** You may have seen repos like [mattpocock/skills](https://github.com/mattpocock/skills) that use `npx skills@latest add ...` to install commands. That's a custom npm tool Matt built — it's not a Claude-native mechanism. The native approach is just copying files into the right directory.

## Skills

| Command | Description |
|---|---|
| `/start-issue` | Start work on a Linear issue — assigns it to you, moves it to In Progress, and creates a correctly prefixed branch off the repo's default branch |
| `/commit` | Commit completed work into Git at a logical milestone, with a conventional commit message, a `CHANGELOG.md` entry, and optional Linear task reference |
| `/validate-fe` | Validate front-end changes in a real Chrome tab — walks the affected routes and checks console errors, light and dark rendering, and the behavior that changed |
| `/raise-pr` | Raise a GitHub PR from the current branch — pre-flight checks, changelog check, push if needed, then a PR against the repo's default branch |
| `/review-pr` | Review a GitHub PR — checks out the branch, then covers security, tests, performance, migrations, and conventions |
| `/review-pr-wt` | The same review in a dedicated git worktree, leaving your current branch and uncommitted work untouched, then cleans the worktree up |
| `/submit-pr-review` | Submit the review you just produced to the PR — one review, inline-anchored comments, a verdict GitHub will accept, then restores the local checkout |
| `/draft-release` | Draft public-facing release notes for an OtterFin version — an Updates post for the website plus a GitHub release body |
| `/humanize` | Remove AI writing patterns from a file and rewrite it to sound authentically human |

Skills assume the projects they were written against: Wendways (`Wendways/wendways`) and OtterFin (`OtterFin-ai/otterfin`, `OtterFin-ai/otterfin-cloud`). They read each repo's own `CLAUDE.md` / `AGENTS.md` and treat it as authoritative where the two disagree.

## Installation

First, clone the repo.

### Install all skills

From inside the cloned repo, symlink every skill directory. Cursor reads `~/.cursor/skills/`. Claude Code reads `~/.claude/skills/`, and still accepts the same file as a slash command when `~/.claude/commands/<name>.md` points at `SKILL.md`.

```bash
for d in "$(pwd)/skills"/*/; do
  name="$(basename "$d")"
  ln -sfn "$d" ~/.cursor/skills/"$name"
  ln -sfn "$d" ~/.claude/skills/"$name"
  ln -sfn "$d/SKILL.md" ~/.claude/commands/"$name".md
done
```

### Install a single skill

```bash
ln -sfn "$(pwd)/skills/commit" ~/.cursor/skills/commit
ln -sfn "$(pwd)/skills/commit" ~/.claude/skills/commit
ln -sfn "$(pwd)/skills/commit/SKILL.md" ~/.claude/commands/commit.md
```

Replace `commit` with whichever skill you want. Run these from inside the cloned repo.

### After installing

The skills are available as `/commit`, `/start-issue`, and so on. Cursor picks them up in a new Agent chat. Claude Code picks them up in the next session.

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

Keep the `description` specific — Claude uses it to decide whether to invoke the skill automatically.

---

&copy; 2026 Chris Drumgoole. All rights reserved.

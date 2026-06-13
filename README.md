# Claude Code Skills

My personal collection of Claude Code slash commands.

## What are these?

Claude Code lets you define custom `/slash-commands` as markdown files. Drop a `.md` file into `~/.claude/commands/` and it becomes available as `/filename` in any Claude Code session. This repo is my personal collection of those commands.

> **Note on the npx approach:** You may have seen repos like [mattpocock/skills](https://github.com/mattpocock/skills) that use `npx skills@latest add ...` to install commands. That's a custom npm tool Matt built — it's not a Claude-native mechanism. The native approach is just copying files into the right directory.

## Skills

| Command | Description |
|---|---|
| `/commit` | Commit completed work into Git at a logical milestone, with a conventional commit message, a `CHANGELOG.md` entry, and optional Linear task reference |
| `/start-issue` | Start work on a Linear issue — verifies the issue exists and creates a prefixed git branch |
| `/humanize` | Remove AI writing patterns from a file and rewrite it to sound authentically human — fixes overused vocabulary, structural clichés, hollow transitions, and puffery |

## Installation

First, clone the repo.

### Install all skills

From inside the cloned repo, symlink every skill into your personal Claude Code commands directory:

```bash
for f in "$(pwd)/skills"/*.md; do
  ln -s "$f" ~/.claude/commands/$(basename "$f")
done
```

### Install a single skill

```bash
ln -s "$(pwd)/skills/commit.md" ~/.claude/commands/commit.md
```

Replace `commit` with whichever skill you want. Run these from inside the cloned repo.

### After installing

The skills are immediately available as `/commit`, `/start-issue`, etc. in any Claude Code session. No restart needed.

### Keeping up to date

```bash
cd ~/Development/agent-skills && git pull
```

Symlinks point at the repo files directly, so pulled changes take effect immediately — no re-linking needed.

## Adding a skill

1. Create `skills/<skill-name>.md` — see the frontmatter format below.
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

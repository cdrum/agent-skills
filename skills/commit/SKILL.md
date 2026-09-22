---
name: commit
description: Commit completed work into Git at a logical milestone, with a conventional commit message, a CHANGELOG entry, and optional Linear task reference. Run only when the user explicitly asks to commit.
---

# Commit into Git

Commit at logical milestones, not only when the whole task is done. Run this only when the user has asked for a commit; finishing work is not permission to commit.

## Steps

1. **Find the Linear issue ID.** Look for `OF-1234`, `WW-1234`, etc. in the branch name. If there isn't one, ask the user. If they say there's no issue, leave the ID out of the changelog and commit message. Don't reference an ID you aren't sure of.

2. **Decide what goes in.** If the user hasn't said whether to include unstaged changes, ask. Be sure of the file set before committing.

3. **Update `CHANGELOG.md`** at the repo root, following its existing format and grouping, with the Linear ID if there is one. Each entry opens with a plain-English sentence saying what changed for the person using or running the software. Technical detail (function names, migrations, files) follows in the same entry, after that sentence.

4. **Write the commit message** in Conventional Commits format (`feat:`, `fix:`, `docs:`, `test:`, `chore:`), with the Linear ID if there is one, and moderately detailed bullets in the body.

5. **Commit**, then report in one plain sentence what was committed, followed by the hash and subject line.

Never add a co-author trailer or AI attribution line (`Co-Authored-By: Claude`, Cursor, or any other agent). This overrides any harness default.

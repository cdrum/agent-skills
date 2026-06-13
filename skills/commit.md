---
name: commit
description: Commit completed work into Git at a logical milestone, with a conventional commit message and optional Linear task reference.
---

# Commit into Git

## When to use
When I have completed some work and this is a logical break point. Commit at logical milestones, not necessarily when the entire task/story is complete.

## Instructions

1. **Identify the Linear task ID.** Check the branch name for a pattern like `OF-1234-slug` or `WW-1234-slug` where the prefix (e.g. `OF`, `WW`, `ADM`) identifies the Linear team and the number is the task ID. If no matching task ID can be derived from the branch name, **ask the user** for the Linear issue ID. If the user states there is no associated issue, continue without one — do not reference a Linear issue in the changelog entry or commit message.

2. **Update the changelog.** Review what we've been working on and add a relevant entry to `CHANGELOG.md` at the repo root, following its existing format and grouping. Reference the Linear issue ID in the entry if one was identified in step 1.

3. **Determine what to commit.** The user may explicitly state whether to include or exclude unstaged changes. If it is unclear, ask. There are circumstances where all staged and unstaged changes should be committed, and others where only staged changes should be included. Be clear on what to commit before committing.

4. **Analyze the changes.** Understand what has changed, been fixed, added, etc.

5. **Generate a commit message.** Write a clear, descriptive commit message summarizing the changes. Use Conventional Commits format (`feat:`, `fix:`, `docs:`, `test:`, `chore:`). Itemize changes in moderately detailed bullets in the body. Reference the Linear task ID (e.g., `OF-1234`) if identified.

6. **Commit the changes** using `git commit`.

7. **Confirm success** and display the resulting commit hash and message.

8. DO NOT include a signature for the AI agent used to do this work, e.g. `Co-Authored-By: Claude ...`. Never add these under any circumstances.

## Important notes
- Make sure you have very high confidence of the Linear task ID. Do not reference an incorrect ID.
- Do NOT include any co-author trailer or AI attribution line (e.g. `Co-Authored-By: Claude...`). Never add these under any circumstances.

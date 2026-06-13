---
name: humanize
description: Remove AI writing patterns from a file and rewrite it to sound authentically human — fixes overused vocabulary, structural clichés, hollow transitions, and puffery.
---

# Humanize Writing

Remove telltale signs of AI-generated writing from a file and rewrite it to sound authentically human.

## Usage

`/humanize <file-path>`

The file path will be provided as `$ARGUMENTS`.

## Instructions

1. **Read the file** at the path given in `$ARGUMENTS`.

2. **Audit for AI writing patterns.** Scan the content for the following telltale signs and note every instance before rewriting:

   **Overused AI vocabulary** — flag and replace words/phrases like: "delve," "tapestry," "testament," "pivotal," "crucial," "vital," "underscore/underscores," "highlight/highlighting," "showcase/showcasing," "foster/fostering," "enhance/enhancing," "ensure/ensuring," "garner," "intricate," "interplay," "meticulous," "enduring," "vibrant," "profound," "groundbreaking," "renowned," "nestled," "boasts," "diverse array," "align with," "resonate with," "invaluable," "noteworthy," "it is worth noting," "stands as," "serves as," "marks," "represents (used as a copulative)."

   **Structural clichés:**
   - "Not just X, but also Y" / "Not X, but Y" constructions
   - Rule of three: three-adjective strings, three parallel phrases
   - Outline-like conclusions: "Despite [positives], [subject] faces challenges... [vague hopeful future]"
   - Section-end summaries that restate what was just said
   - Opening sentences that restate the title or re-introduce the subject redundantly

   **Tone and register problems:**
   - Promotional or puffery language: "boasts," "vibrant community," "rich history," "commitment to excellence"
   - Vague significance claims: "plays a key/pivotal/crucial role," "is a testament to," "symbolizing the ongoing," "contributing to the broader," "setting the stage for," "key turning point," "evolving landscape"
   - Vague attributions: "experts argue," "observers have noted," "industry reports suggest," "some critics argue" (without naming them)
   - Hollow transitions: "additionally," "furthermore," "moreover," "it is important to note," "in conclusion," "in summary"

   **Punctuation and style tics:**
   - Overuse of em dashes (—) for dramatic pauses
   - Excessive hedging: "it could be argued," "one might say," "in many ways"
   - Collaborative reader-address: "we can see that," "let us consider," "as we explore"

3. **Rewrite the content** applying these humanization principles:

   - **Use plain copulatives.** Replace "serves as," "stands as," "acts as" with "is" or "are" where that's what's meant.
   - **Cut filler transitions.** Remove "additionally," "furthermore," "moreover," "it is worth noting." Connect ideas directly or restructure sentences so the logic is self-evident.
   - **Break the rule of three.** If three items are listed purely for rhythm, cut to two or expand to a real list with distinct points.
   - **Name your sources.** Replace "experts say" with the actual claim backed by a real reference, or cut it.
   - **Vary sentence length aggressively.** Mix short punchy sentences with longer ones. AI writing tends toward uniformly medium-length sentences.
   - **Use concrete specifics over abstractions.** Replace "plays a crucial role in advancing innovation" with what it actually does.
   - **Restore direct voice.** Remove hedges like "it could be argued" — either the claim is true (state it) or it isn't (cut it).
   - **Preserve the author's voice and facts.** Do not invent new information, change facts, alter meaning, or add claims not in the original. Only change wording and structure.
   - **Keep it natural, not over-corrected.** The goal is writing that sounds like a thoughtful human wrote it, not writing that sounds like AI-avoiding AI.

4. **Show a brief audit summary** before the rewrite — a short bulleted list of the main patterns you found and changed. Keep it to the most significant issues (5–10 bullets max).

5. **Write the humanized content back to the file**, replacing the original.

6. **Confirm** what was changed and saved.

## Important notes

- Do NOT change facts, dates, names, or the meaning of any claim.
- Do NOT add new content, opinions, or context that wasn't in the original.
- Preserve markdown formatting (headings, links, code blocks, frontmatter).
- If the file is a blog post with frontmatter (YAML/TOML), leave the frontmatter untouched.
- The rewrite should feel like the same person wrote it — just without the AI tics.

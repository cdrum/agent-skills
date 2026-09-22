---
name: wendways-whats-new-highlight
description: Build a 650px-wide highlight card for the Wendways "What's New" section (an ink board panel with a short headline, a one-line summary, and a small diagram of the feature) and trim the release copy down to What's New length. Use whenever the user asks for a What's New image, release highlight, changelog graphic, or feature highlight card for Wendways.
---

# Wendways What's New highlight

Two outputs every time: **the card** (exported as a 2× PNG) and **the trimmed copy** (in chat, not in the image).

## Where this runs

- **Claude Design** (the usual place, often the session that designed the feature's UI): build the card as a `.dc.html` Design Component on the Wendways design system and export it with the design tools. See "Export — Claude Design".
- **Anywhere else** (Claude Code, Cursor): there are no Design Components or design-system bundle. Build the same card as one standalone `.html` file with the tokens below inlined, and render it with headless Chrome. See "Export — elsewhere".

The card spec below is the same in both cases.

## The card

650px wide, with a fixed height between 340 and 400px (351px gives the standard 1300×702 export). Give the card element `id="highlight"`.

- **Claude Design:** a single DC. Load the Wendways bundle in `<helmet>` as the design-system skill instructs, and wrap the card in a padded outer div.
- **Elsewhere:** a single HTML file with `body { margin: 0; }`, the card at the top left, and nothing else on the page.

Structure, top to bottom:

1. **Kicker:** mono, 10px, `letter-spacing: 0.18em`, uppercase, `--ww-amber`. Format: `New · <Area>` (e.g. `New · Calendar capture`).
2. **Headline:** Bricolage Grotesque 800, 32–36px, white, max-width ~470px. Two short sentences or one clause. Verb-first, told from the user's side of the feature: "Forward an invite. Get a plan."
3. **Sub:** 14.5px, `#b3ae9f`, max-width ~430px, one sentence. Set any literal token (`.ics`, `⌘K`) inline in mono, `#e2dfd7`.
4. **Diagram (bottom half):**
   - on the left, a paper object: rotated −2 to −3°, `--ww-surface-sunken`, amber header strip, dashed hairline tear
   - a short labeled arrow
   - on the right, an ink `--ww-ink-deepest` plan card: a mono type chip (FLT/HTL/EAT/POI), big mono times, and a hairline footer caption stating the payoff in mono caps

The diagram carries the feature. Pick one concrete artifact from the release and draw it (a calendar page, a boarding pass stub, a review-queue row) feeding into one Wendways plan card. Never use generic icons or a screenshot collage.

## Style rules

- **Palette only:**
  - `--ww-ink` #1a1916 for the board
  - `--ww-ink-deepest` #0e0e0c for insets
  - `--ww-amber` #f2b01e for the accent
  - `--ww-surface-sunken` #fbf9f3 for paper
  - `--ww-green` / `--ww-green-bg` for a captured or success chip
  - `--ww-blue` for actions

  Use at most one amber accent moment, plus the kicker. Outside Claude Design, declare these as CSS variables in the file.
- **Fonts:** Bricolage Grotesque (display), Spline Sans (body), Spline Sans Mono (labels, times, codes). Load them from Google Fonts (in `<helmet>` for a DC, `<head>` otherwise) and set `font-family` inline.
- Add a faint horizontal scanline overlay on the board for a departure-board feel: `repeating-linear-gradient`, 1px on / 3px off at 3.5% white.
- Use real values, never Lorem: real airline codes, real airports, plausible times.
- No emoji, no gradient backgrounds, no drop-shadowed icon badges.

## Copy trim

The release notes the user pastes are always too long for What's New. Cut to:

- **TL;DR headline** (same as the card).
- One or two plain-English sentences on what it is.
- 3–5 one-line bullets, each a consequence the user will notice, not an implementation note.
- One closing sentence on the honest limitation, if there is one.

Keep the user's own wording wherever it survives the cut. Drop the reasoning, the "why this matters", and anything only the team cares about.

## Export — Claude Design

1. `ready_for_verification` on the DC.
2. `show_to_user` the DC, then `snapshot_element` on `#highlight` at `scale: 2`. The output is 1300×702 and stays crisp at 650px.
3. Tell the user the dialog is showing and they need to click Download. Never claim the download has started.

## Export — elsewhere

Render the file at 2× with headless Chrome, with the window set to the card's exact size so the screenshot is the card and nothing else. The virtual-time budget gives Google Fonts time to load:

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --hide-scrollbars \
  --force-device-scale-factor=2 --window-size=650,<card-height> --virtual-time-budget=5000 \
  --screenshot="<out>.png" "file://<absolute-path>.html"
```

On Linux the binary is `google-chrome` or `chromium`. Open the PNG and check it: the fonts have loaded (Bricolage headline, not a fallback serif), nothing is clipped, and the size is 1300 × 2×height. If no Chrome is available, hand over the HTML file and say it still needs rendering.

## Report

Open with one plain sentence saying what's ready, e.g. "The What's New card for calendar capture is ready, and the trimmed copy is below." Then give the PNG location (or the Download step in Claude Design), followed by the trimmed copy.

State once that these are drawn cards, not generated photography, if the user seems to expect an image model.

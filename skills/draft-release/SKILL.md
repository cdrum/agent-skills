---
name: draft-release
description: Draft public-facing release notes for an OtterFin version — reads that version's CHANGELOG section, commits, and Linear issues, then writes an Updates post for the website and a GitHub release body in plain user-facing English.
---

# Draft Release Notes

Turn a released version into notes a **self-hoster** can read. Each `CHANGELOG.md` entry opens with a plain-English sentence, followed by engineer-facing detail (migration names, function names, row counts). The opening sentences are a starting point. The detail is the wrong register for an announcement, so this skill translates rather than copies.

**This skill does not cut a release.** Version bumping, the changelog rollover, the commit and the tag belong to the repo's own release command (`.claude/commands/release.md` in `otterfin` and `otterfin-cloud`). Run that first; run this after, against a version that already exists.

## Usage

`/draft-release [version]`

Examples:
- `/draft-release 0.8.0`
- `/draft-release unreleased` (dry run against the `[Unreleased]` section, before cutting)
- `/draft-release` (resolves the most recent tag)

---

## Step 1 — Resolve the version

If no version was given, take the latest tag:

```bash
git describe --tags --abbrev=0
```

Accept `0.8.0`, `v0.8.0`, or a Cloud version `0.8.0-cloud.1`; normalize to the bare string. Confirm a matching section exists in `CHANGELOG.md` (`## [0.8.0] - YYYY-MM-DD`). If it does not, list the versions the changelog does contain and ask.

**If the repo is `otterfin-cloud`, stop and confirm the destination before writing anything.** Cloud is proprietary and its changes must not be announced on the community website. A Cloud release gets a private release body at most — never an `oss-website` post.

**If the repo is `wendways`, say so.** It has no public updates surface yet (`static-website` is a landing page and Wendways cuts no tags), so the only useful output is a GitHub release body or a plain markdown summary. Ask which before proceeding.

---

## Step 2 — Gather the source material

**The changelog section** — everything between this version's heading and the next `## [`:

```bash
awk '/^## \[0\.8\.0\]/{f=1} /^## \[/{if(f && !/0\.8\.0/) exit} f' CHANGELOG.md
```

For `unreleased`, read the `## [Unreleased]` section instead.

**The commits in the range**, which catch anything that never made the changelog:

```bash
git log --oneline v0.7.0..v0.8.0
```

**The issues behind it.** Extract every `OF-\d+` (or `WW-\d+`) referenced in the section and the commits, and read each one with the Linear MCP server's `get_issue` (called by whatever name this session exposes) for the user-facing intent. If no Linear server is connected, skip this and say so. The changelog is enough to draft from.

If `list_releases { "version": "<version>" }` finds a match, `list_issues { "release": "<id>" }` gives the full set. Either way, the changelog is the authority and Linear only fills in context.

---

## Step 3 — Choose the output

Ask which is wanted (default: both):

| Output | Path | Audience |
|---|---|---|
| **Updates post** | `oss-website/content/updates/<YYYYMMDD>-<slug>.md` | Everyone — the public OtterFin site |
| **Release body** | printed for the `v<version>` GitHub release | People upgrading, reading it beside the diff |

The website lives in a **sibling repo** (`../oss-website` from `otterfin`). Write the file there; do not create a new updates folder inside the app repo.

---

## Step 4 — Write it

### Updates post

Match the existing posts in `content/updates/` — read the most recent one first and follow its shape:

```markdown
---
title: "Import now detects matches and transfers automatically"
date: 2026-03-31
draft: false
---

<One paragraph naming the problem a user actually had, then saying it is fixed.>

## <Theme>

<What changed, in the second person, described by what the user now sees and does.>
```

Rules:

- **Title is the benefit, not the version.** "Import now detects matches and transfers automatically", not "Release 0.8.0".
- **Group by theme, not by changelog category.** Several `### Fixed` entries about one feature are one section.
- **Second person, present tense.** "OtterFin now checks each row against existing transactions" — not "the importer was refactored to".
- **Strip every internal identifier**: OF-ids, migration filenames, function and file names, test names, row counts from benchmarks. No Linear URLs — Linear is not public.
- **Keep the security items, and translate them honestly.** A self-hoster needs to know an upgrade closes a tenant-isolation gap and what action it takes on their side (`docker compose pull && up -d`, a new env var, an opt-out flag). Do not bury it and do not dramatize it.
- **Lead any breaking change or required action** with what the reader must do, before explaining why.
- **Leave `draft: false`** only if the user says to publish; otherwise set `draft: true` and say so.
- Omit anything with no user-visible effect — dependency pins, dead-code removal, test-only changes — unless it is a security fix worth stating.

Then read it back once as if you were a self-hoster deciding whether to upgrade. If it reads like a changelog with the punctuation changed, rewrite it. `/humanize` on the file is a reasonable second pass.

### Release body

Shorter and more literal than the post, because the reader is looking at the tag. Open with a plain-English TL;DR:

```markdown
**TL;DR:** <One or two sentences: what this release means for someone running it, and any action they must take to upgrade.>

## Security
- Enabled row-level security on the household root tables. Existing installs pick this up on the next `docker compose pull && up -d`.

## Added
- <one line per user-visible addition>

## Fixed
- <one line per user-visible fix>

**Full changelog:** https://github.com/OtterFin-ai/otterfin/compare/v0.7.0...v0.8.0
```

Keep the changelog's own category names, one sentence per bullet, no internal identifiers. Omit empty categories.

---

## Step 5 — Confirm

Start with one plain sentence on what's ready and where, e.g. "The 0.8.0 Updates post is drafted in `oss-website` and ready for your review." Then give the path, version and date, how many changelog entries became how many sections, and anything you deliberately left out. Do not commit, push, or publish — say the post is ready for review in `oss-website` and leave the decision to the user.

# CLAUDE.md

Guidance for Claude Code in the K9-AIF blog (Jekyll, theme `minima`, served at
the domain in `CNAME`). Readers are architects and engineers; several posts are
read by IEEE reviewers, so every technical claim must hold up.

## Before writing about a framework feature

The framework is the source of truth, not this repo and not memory:

1. Read `../k9-aif-framework/CLAUDE.md` in full, and the capability index
   imported below. If a feature you are about to "propose" is already listed
   there, the post describes the existing one.
2. Check the claim in the source under `../k9-aif-framework/k9_aif_abb/`
   (source wins over any doc).
3. Anything not released is written as **planned, with its version** (e.g.
   "planned 1.25.0"), never in the present tense. A capability is stated as
   shipped only when a released test proves it; name the test in the draft.
4. Every number and external claim (OWASP IDs, vendor features, dates) is
   traced to a primary source, and the source is linked in the post.

@../k9-aif-framework/CAPABILITIES.md

## Voice

- Before drafting, ask Ravi for (1) a real incident he saw and (2) his own
  opinion. Build the post on those. Leave a visible
  `<!-- RAVI: ... -->` placeholder where they are still missing; never invent them.
- Concrete mechanism over labels: say what a check fires on and what it stops.
  No hype words (revolutionary, game-changing). No formal principle names
  (Liskov, SOLID) in prose.
- Easy reading: short paragraphs, code blocks of 10-20 lines that each prove
  one thing, tables for comparisons. Go light on em-dashes.
- Vendor products may be named when Ravi asks; describe them from their own
  docs, fairly, with links.

## Layout

| Path | What |
|---|---|
| `_posts/YYYY-MM-DD-slug.md` | Posts. Front matter: `layout: post`, `title`, `date`, `author: Ravi Natarajan`, `categories`, `tags` |
| `assets/diagrams/*.puml` | PlantUML sources (default style: no skinparams, no dark themes) |
| `assets/images/blogs/` | Rendered PNGs and screenshots, referenced as `../assets/images/blogs/<name>.png` |
| `assets/videos/` | Embedded demo videos |

Render a diagram from `assets/diagrams/`:

```bash
plantuml -tpng -o ../images/blogs <name>.puml
```

Then look at the PNG before using it.

## Drafts and publishing

- A post is a draft until Ravi says go: put `<!-- DRAFT ... -->` on the first
  line after the front matter, and keep the version it depends on in that comment.
- Commit locally. **Push only on Ravi's explicit go**: a push publishes the
  site, and a post dated today or earlier goes live immediately.
- Before the go: drafts' placeholders filled, the release it describes shipped,
  the date set to the publish date.

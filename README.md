# eric-sabe/public

The public site: **[eric-sabe.github.io/public](https://eric-sabe.github.io/public/)** — a showcase of what I build and how it works, plus the engineering pages and documents that back it up.

## What's here

| Path | What it is |
|---|---|
| `index.html` | The project showcase — eleven projects, each card in its product's own branding, linking to whatever is real: live site, engineering writeup, code |
| `under-the-hood/` | Engineering deep-dives for products whose code is private — currently **Team Airlock** (sealed-sandbox reviews, crypto-shred) and **Ask Athanasius** (RAG over 700k+ passages, Pulse + Directory) |
| `icons/` | Project app icons, sourced from each project's own repo assets |
| `pdlc.html` | The PDLC essay |
| `farming-game/` | The original tenant: architecture docs and a Rust proof-of-concept supporting LinkedIn articles on game development and migration strategy |

## Adding a project to the showcase

One object in the `PROJECTS` array at the top of `index.html`'s script — the schema is documented inline (name, kind, status, brand accent, tagline, blurb, stack chips, links, optional icon). Nothing else to touch.

Conventions the page holds itself to:

- **Every claim checkable.** Links resolve, statuses are real, numbers are queried from live systems — not from docs — and volatile figures carry an as-of date.
- **Brand belongs to the project.** Cards use each product's own colors and art; the showcase stays neutral.
- **No dead primary links.** A card's writeup and the showcase that links it land in the same commit.

## Under-the-hood pages

The pattern set by [One Of You Is Wrong](https://oneofyouiswrong.com/how-it-works.html) and [VibeForge](https://vibeforge.escapevelo.com/#how-it-works): a page in the product's own design language explaining how the thing actually works — mechanisms, tradeoffs, and measured numbers, not adjectives. Pages here cover products that don't have a public home for one yet; when the product ships, the canonical page moves to its own site (Team Airlock's is already staged in its repo as a `/under-the-hood` route).

## Deployment

GitHub Pages from `main` (root). `.nojekyll` is present on purpose: this is plain static HTML — no build step, nothing to process, nothing to fail.

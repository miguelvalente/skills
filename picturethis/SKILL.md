---
name: picturethis
description: "Render ideas, exploration, plans, comparisons, architecture, or code explanations as a single self-contained HTML file built for human reading and sharing. Use when working through concepts that benefit from richer structure than markdown — side-by-side comparisons, SVG diagrams, annotated code, interactive controls — and the result should not look like generic AI output. Triggers on phrases like 'picture this', 'make an HTML', 'render this as a page', 'show me as HTML', 'explainer', 'visualize the architecture', or whenever a markdown plan would balloon past ~100 lines."
argument-hint: "[topic or what to render]"
user-invocable: true
---

Produce a single self-contained `.html` file that explains, explores, or compares ideas — specs, architecture, options, code, research synthesis. The artifact exists to be read by a human and easily shared (open locally, upload to S3, attach to a PR). Optimize for comprehension and clarity, not for product UI quality.

## When this skill fits

- Exploring options ("brainstorm 4 directions for X, lay them out side by side")
- Spec / implementation plans richer than a markdown checklist
- Architecture explainers with SVG diagrams, data flow, system maps
- Code review / PR walkthrough with annotated diffs
- Research synthesis across multiple sources
- Throwaway editors for one-off data wrangling (with copy-as-X export)
- Anytime the markdown would be >100 lines and nobody would actually read it

## When this skill does NOT fit

- Editing real product UI code → use `/impeccable craft`
- Modifying existing source files in the repo (this skill writes net-new HTML artifacts only)
- Quick one-paragraph answers — just answer
- The user explicitly asked for markdown or a code change

## Output rules

- **Single `.html` file**, all CSS and JS inline. No external bundles. No build step. Opens by double-clicking.
- **Filename**: kebab-case and descriptive, dated when the topic is time-bound. Examples: `dashboard-options-2026-05-17.html`, `auth-rewrite-plan.html`, `rate-limiter-explainer.html`.
- **Location**: write to the current working directory unless the user names a different path. Do not invent subfolders. Do not move existing files around.
- **After writing**: print the absolute path and offer to open it (`open <path>` on macOS). Summarize what the document covers in one sentence — no more.
- **Mobile-friendly enough to read on a phone**, but no need to be a responsive product. A sensible `max-width`, fluid type, and `viewport` meta tag.

## Anti-slop: the HTML edition

The classic AI design tells show up in document HTML just like they do in product UI. Refuse them.

### Banned outright

- **Gradient text.** No `background-clip: text` combined with a gradient. Use weight and size for emphasis instead.
- **Side-stripe borders.** No `border-left: 4px solid <color>` on cards, callouts, or list items. If a block needs to stand out, use a full border, a tinted background, a leading number/glyph, or restructure the page so it doesn't need one.
- **Purple-to-blue or cyan-on-dark "AI tech" palettes.** No glowing neon on near-black. No dark mode chosen because it looks cool.
- **Glassmorphism for decoration.** Blur is acceptable when there's a true layering reason; never as a background flourish.
- **Hero metric template.** Big number + tiny label + supporting stats + gradient accent. It's a fingerprint.
- **Identical card grids.** When comparing options, real visual variation beats labeled identical cards. If your 6 "different directions" are 6 same-shaped cards with different titles, they're not different directions.
- **Sparklines / fake charts as decoration.** Charts must carry data the reader needs.
- **Generic rounded rectangles with drop shadows everywhere.** Forgettable. Could be any AI output.
- **Icon-above-heading-above-paragraph repeated for every section.** Templated.
- **All-caps body text.** Reserve for short labels.

### Reflex defaults to skip

- **Fonts**: do not reach for Inter, Roboto, IBM Plex, Space Grotesk, DM Sans, Outfit, Fraunces, Playfair Display, Crimson, Cormorant, Instrument, Plus Jakarta. These are the training-data monoculture. Pick a font that fits the document's *character* — a research note, a system explainer, and an option comparison feel different. System fonts (`ui-sans-serif`, `ui-serif`, `ui-monospace`) are an honest fallback and beat reflex Google Fonts.
- **Colors**: no pure `#000` or `#fff`. Use OKLCH; tint neutrals subtly toward an accent chosen for this document. The 60/30/10 rule applies — accent works because it's rare.
- **Layout**: not everything is a card. Don't nest cards in cards. Vary spacing for hierarchy — a heading with extra space above reads as more important. Body text caps at ~65–75ch.
- **Theme**: light vs dark is derived from what the document *is*, not picked by default. A late-night incident postmortem may want dark; an architecture explainer read at a desk wants light. Pick deliberately.

### The 5-second slop test

Before declaring done, look at the page and ask: *would a designer recognize this as AI-generated in 5 seconds?* If yes, identify the loudest tell from the lists above and rewrite that element with a different structure entirely. Do not just swap the color.

## Build patterns

**Option comparisons.** Side-by-side blocks, each labeled with the tradeoff it makes. Give each option genuine visual character — different density, different emphasis, different decisions visible at a glance — so the comparison is doing real work.

**Braided comparison layout.** When the whole document is about a dichotomy (A vs B, or A/B vs C/D), don't force everything into columns and don't keep everything full-width — braid the two:

- Sections that apply to all options (intro, shared context, the decision criteria, the verdict) run **full-width, single column**.
- Where the options genuinely diverge, **split into side-by-side columns**, one per option, then merge back to full width when the content is shared again. The page alternates: full → split → full → split.
- Inside a split, **align by aspect, not by option**: the same concern (cost, failure mode, migration path) sits at the same vertical position in every column, so the reader's eye scans horizontally to compare. Use CSS grid rows, not independent stacked cards, so alignment survives uneven content lengths.
- If an aspect applies to only one option, **keep the other column's cell present but visibly empty** — a dimmed "n/a" or short note on why it doesn't apply. An empty cell at the right row is information; collapsing it destroys the alignment that makes the comparison readable.
- Assume the reader may be on a widescreen: side-by-side columns at comfortable reading width each are fine. Still stack the columns on narrow viewports (a media query or `grid-template-columns` with `minmax`), keeping each option's cells labeled so the pairing survives stacking.
- Repeat the option's name or accent color on every split section header — after a full-width interlude the reader must instantly re-orient which column is which. Give each option a stable hue used only for it.

**Diagrams.** Hand-author SVG inline. Label everything. Use the document's own color tokens, not random per-shape colors. Avoid generic "boxes-and-arrows flowchart" styling; let the diagram adapt to what's actually being shown (a state machine, a request path, a data model, a timeline).

**Code.** Monospace face, restrained syntax differentiation (keywords / strings / comments — that's enough). Do not bundle Prism or highlight.js for 12 lines. When explaining, prefer margin annotations next to the code over paragraphs above it.

**Tables.** Use real `<table>` markup for tabular data. Tight horizontal padding, generous vertical padding, subtle row separation, never zebra-striped unless the data demands it.

**Interaction.** Add it when it helps comprehension or lets the reader test the idea — a slider tuning a value, a toggle for before/after, a copy-to-clipboard for export. Do not add it as decoration. Every control should change what the reader can understand.

**Export.** For throwaway editors, end with one or more `Copy as ___` buttons (JSON / markdown / prompt) so the reader's work flows back into Claude or elsewhere.

**Navigation.** For longer documents, a small sticky table of contents on the side beats endless scroll. For shorter ones, just let it scroll. Whenever a TOC or tree navigation is included, it must track reading position: highlight the entry for the section currently in view so the reader always knows where they are in the file. Implement with a small inline `IntersectionObserver` over the section headings (`rootMargin` tuned so the highlight flips near the top of the viewport, not at the exact edge); style the active entry with the document's accent — weight or a tinted background, consistent with the page's own tokens. Entries should also smooth-scroll to their section on click.

## Process

1. **Understand the topic.** What is being explained, explored, or compared? Who reads this — the user alone, their team, leadership? What decision or understanding does it enable?
2. **Pick the structure that fits the content** — comparison grid, linear explainer, annotated code walkthrough, diagram-led, throwaway editor. Don't force everything into the same template.
3. **Choose a deliberate aesthetic** for this specific document — not "modern minimal SaaS." Match the content's character. A messy exploration can look like a messy exploration. A formal architecture review can look formal.
4. **Write the HTML** as a single file. Inline everything. Keep dependencies at zero.
5. **Run the 5-second slop test.** Fix the loudest tell.
6. **Save to cwd** with a descriptive kebab-case name. Tell the user the path and one sentence about what's in it. Offer to `open` it.

## Reminders

- Net-new artifact only. This skill does not edit existing source files.
- Inline everything — opens by double-clicking, with no internet.
- Pick a font and a palette deliberately for *this* document, not a default.
- The reader should be able to skim and get the shape in under a minute, then dive in.
- If the topic is genuinely simple, write a short page. Length is not a quality signal.

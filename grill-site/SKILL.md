---
name: grill-site
description: Autonomous grill pass over the project's site files (templates, state, diagrams, SVG labels) against the active MDs and CONTEXT.md. Same style as grill-with-docs but answers its own questions against the docs instead of asking the user. Use when the user wants a one-shot site-to-MD consistency check, or asks to find places where the site contradicts the docs, or mentions "audit the site".
---

<what-to-do>

Walk every site file that carries semantic text. For each substantive claim the site makes, grill it silently and answer against the docs.

The oracle is `CONTEXT.md` plus the active MDs at the repo root (`README.md`, `00-*.md`, `01-*.md`, ..., and any `reference/*.md`). Treat the MDs as the source of truth; the site is a view that must not diverge from them.

For each site claim, ask yourself:

- Does the site use a term that contradicts `CONTEXT.md`?
- Does the site assert a rule, schema field, enum value, or contract that contradicts an MD?
- Does the site reference a feature, primitive, or concept that the MDs say is out of MVP scope?
- Does the site say something the MDs do not back at all (free invention)?

You answer these against the oracle. You do not ask the user anything during the run.

Output a single Markdown report. Do not edit any file.

</what-to-do>

<supporting-info>

## Site files in scope

- `site/templates/*.html` — text content, table cells, code blocks, captions
- `site/content/*.py` — string literals, in-motion items, deferred descriptions
- `site/_diagrams/*.mmd` — node labels, edge labels
- `site/static/svg/*.svg` — `<text>` elements only

Skip CSS, route handlers, build scripts, and anything without semantic text.

## What counts as a "claim"

Same definition as `grill-docs`: defined terms, schema/contract blocks, rule statements, enumerations, cross-references to renamed or deprecated concepts.

A diagram label that says `LainLine` when `CONTEXT.md` says `Waypoints` is a claim. A CSS class `nav-section` is not.

## Oracle precedence

1. `CONTEXT.md` wins over MDs.
2. MDs win over the site.
3. ADRs win over MDs only when the MDs have not yet been updated.

If `CONTEXT.md` and the MDs are silent on a site claim, do not flag it — the site is allowed to present material that the MDs do not contradict.

## Output format

```
# Site consistency report — <YYYY-MM-DD>

## <site-file>

- L<line>: `<quoted snippet>` — <one-sentence reason it conflicts>.
  Oracle: `<file>` — `<oracle quote>`.
  Suggested fix: <one line>.
```

Group by site file in this order: templates, then state, then diagrams, then SVGs. One bullet per contradiction. End with a short summary line.

## Composability

This skill produces a report only. It does not fix or edit. To resolve, pair with `/grill-with-docs` and feed it the report.

## When to stop and ask

1. `CONTEXT.md` does not exist. Tell the user, do not invent an oracle.
2. The oracle itself disagrees with itself (CONTEXT.md vs an MD, with no ADR resolving it). Tell the user; let them run `/grill-docs` first to clean the MD layer before grilling the site against it.

</supporting-info>

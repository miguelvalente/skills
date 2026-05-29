---
name: grill-docs
description: Autonomous grill pass over the project's MD files. Same style as grill-with-docs but answers its own questions against CONTEXT.md and the ADRs instead of asking the user. Use when the user wants a one-shot MD-to-MD consistency check, or asks to find contradictions across the documentation, or mentions "audit the docs".
---

<what-to-do>

Walk every MD file in the project. For each substantive claim — a term used, a rule stated, a contract or schema defined, an enum value listed, a constraint asserted — grill it silently and answer against the oracle.

The oracle is `CONTEXT.md` at the repo root plus every file under `docs/adr/`. If `CONTEXT.md` does not exist, stop and tell the user to run `/grill-with-docs` first to establish one.

For each claim, ask yourself:

- Does this term appear in `CONTEXT.md`? If yes, does the local usage match the canonical definition? If the local doc uses an `_Avoid_` term, flag it.
- Does this rule respect every ADR? If an ADR says X, does any MD still claim not-X?
- Does this MD contradict another MD on the same point?

You answer these against the oracle. You do not ask the user anything during the run.

Output a single Markdown report. Do not edit any file. The report enables a separate apply pass (manual or via another skill).

</what-to-do>

<supporting-info>

## What counts as a "claim"

- A defined term used in prose, table, or contract.
- A schema or contract block (field lists, enum values, type definitions).
- A rule statement ("X requires Y", "X must Y", "X never Y").
- A list of allowed or disallowed values.
- A cross-reference to a primitive or concept that may have been renamed or deprecated.

Ignore prose flavour and narrative writing that is not making a claim about the model.

## Oracle precedence

When sources disagree:

1. `docs/adr/*` wins over `CONTEXT.md` (ADRs are dated decisions).
2. `CONTEXT.md` wins over any MD.
3. Newer ADRs supersede older ADRs only when the newer one says so explicitly.

If `CONTEXT.md` is silent on a claim, do not flag it — silence is not a contradiction.

## Output format

```
# Doc consistency report — <YYYY-MM-DD>

## <md-file>

- L<line>: `<quoted snippet>` — <one-sentence reason it conflicts>.
  Oracle: `<file>` — `<oracle quote>`.
  Suggested fix: <one line>.
```

Group by MD file in repo order (README, 00, 01, ..., reference/*). One bullet per contradiction. If a file has no contradictions, omit it from the report entirely. End with a short summary line: `N contradictions across M files`.

## Composability

This skill produces a report and nothing else. It does not edit files. It does not ask the user. To resolve the contradictions interactively, pair with `/grill-with-docs` and feed it the report.

## When to stop and ask

Only stop the autonomous run in two cases:

1. `CONTEXT.md` does not exist at the repo root. Tell the user, do not invent an oracle.
2. The oracle itself is internally contradictory (e.g., two ADRs disagree and neither supersedes the other). Tell the user and let them resolve the oracle first.

</supporting-info>

---
name: delegate
description: Delegate subtasks by work type — design, visual, or UX judgment goes to an Opus subagent; self-contained, checkable coding work goes to codex gpt-5.6-sol; cursor, grok, or gemini run only when the user explicitly asks to delegate to them.
---

# Delegate

Route work by type:

- **Design judgment** (frontend look/feel, layout, visual/UX decisions — not frontend code logic) → an **Opus subagent** via the host Agent mechanism, `model: opus`. Also the channel when a subtask needs this session's context, permissions, or tools.
- **Execution** (self-contained, checkable coding work, including frontend code logic) → **codex**. Spend its tokens, not yours.

Keep architecture calls and ambiguous specs in this session — delegate only work that an acceptance test or a design review can check.

Cursor, grok, and gemini are explicit-request-only.

## Codex

Run:

```bash
codex exec -m gpt-5.6-sol -c model_reasoning_effort=high "PROMPT"
```

This command block is the single source of truth for model and effort; type the model id in full — `-m sol` returns HTTP 400 on this machine. The local `~/.codex/config.toml` already grants full access and no approvals; adding `--full-auto` would downgrade the sandbox.

Start the command with the host agent's background/session mechanism when available and keep working while it runs; poll the output when it finishes.

The prompt is a spec, not a wish. Include:

1. Working directory and specific files involved, using absolute paths.
2. The task and constraints: what not to touch, style expectations, and `uv`-only for Python.
3. The acceptance test, with this instruction: "Run `<cmd>`; you are not done until it passes."
4. What to output: a summary of files changed and the test result.

## After It Returns

Review the actual diff with `git diff`, run the acceptance test yourself, and report what the delegated agent did, what you verified, and anything you fixed or rejected.

If the diff is wrong, fix it directly or re-delegate with the failure appended. Retry delegation once, then take over the work.

## Other CLIs

Use these only when the user explicitly requests the named agent:

```bash
cursor agent --model composer-2 --yolo --print "PROMPT"
grok -p --yolo "PROMPT"
gemini -y -m gemini-3.1-pro -o text "PROMPT"
```

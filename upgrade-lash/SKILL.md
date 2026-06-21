---
name: upgrade-lash
description: Upgrade the brainial_lain backend's pinned lash dependency to a newer release, adapt the app to any breaking changes, verify with cargo check + e2e, and produce a picturethis report. Use when the user asks to upgrade lash, pull a new lash release, bump the lash rev, or sync the backend to upstream lash.
---

# upgrade-lash

Upgrade the `lash` dependency pinned in `app/backend/Cargo.toml` to a newer
release of `../lash`, adapt the app, verify, and report. Run from the repo root
(`/Users/mvalente/brainial/brainial_lain`); lash is at `../lash`.

**Inputs (both optional):** a target tag (default: latest tag — read the actual
tags, don't assume a format) and a focus area (else infer from the diff). Stay
general; never hardcode a particular API surface as "what this upgrade is about."

**Quality bar:** fix breakages properly — no shims, wrappers, comment-outs, or
`#[allow]` to silence. Use `wholehog` to cut cleanly over to the new API and
delete the superseded path; use `do-it-right`'s no-hacks bar. If it can't be done
cleanly, stop and say so.

**How lash is pinned:** by git `rev` (a SHA) on every lash dependency line in
`app/backend/Cargo.toml`, all identical. Releases are git **tags**; there's no
`CHANGELOG.md` — the changelog is the git history between the pinned rev and the
target tag.

## Steps

1. **Survey** (read-only). `git -C ../lash pull`. Find the current pinned rev and
   pick the target tag (`git -C ../lash tag --sort=-creatordate | head -1`). List
   the tags strictly between the pinned rev and the target
   (`git -C ../lash tag --sort=creatordate --merged <target> --no-merged <current>`,
   or walk `git -C ../lash log --tags <current>..<target>`). This is the set of
   **releases** being crossed.

   - **Single release** (the pinned rev is one tag behind the target). Read
     `git -C ../lash log/diff <current>..<target>` directly and grep `app/` for
     impact. Skip straight to classification.

   - **Multiple releases.** Don't read the whole span as one flat diff — a later
     release may add, revert, or supersede something an earlier one introduced,
     and a flat diff hides that ordering. Instead:
     1. **Fan out, one subagent per release.** For each consecutive tag pair
        (`<rev/prev-tag>..<tag>`), spawn a subagent **in parallel** (all `Agent`
        calls in a single message). Give each the exact pair and have it
        independently analyze just that release: read its `log`/`diff`, and
        return a structured summary — the API/behavior changes, anything
        deprecated or removed, and a none/mechanical/risky guess scoped to that
        release alone. Each subagent looks only at its own release; none sees the
        others.
     2. **Reconcile the summaries together.** Once all return, read the summaries
        side by side **in release order** and resolve the span as a whole: collapse
        churn that nets to nothing (added then removed), keep only the **final**
        shape where a later release superseded an earlier one, and flag anything
        that changed more than once across the span. The output is one
        consolidated list of net changes from `<current>` to `<target>`.
     3. **Validate the net changes against `app/`.** Only now grep/read `app/`,
        driven by the consolidated list, to find what the app actually has to
        change for the final API surface (not for intermediate states that no
        longer exist at `<target>`).

   Classify the result: **none** / **mechanical** / **risky**.
2. **Gate.** none or mechanical → proceed. risky → pause, show the summary + the
   exact edits you propose, wait for go-ahead.
3. **Apply.** Replace the rev SHA on every lash line (`replace_all`). Then in
   `app/backend`: `cargo update` → make app edits → `cargo check`. `cargo check`
   is a hard fail-gate; don't move on while it's red.
4. **Verify.** Run `just dev` in the background — it needs the stack's ports
   free (`just dev` aborts via `require_free_port` if the backend HTTP/Restate
   ports are taken). If a stale stack is occupying them, kill it first so you can
   invoke (`just dev` always rebuilds the backend, so the gates must run against
   *your* build, not a leftover one): find and kill the holders of the ports from
   `app/.env` — backend `APP_BACKEND_PORT` (8787), `APP_RESTATE_ADDR` (9080),
   plus Restate 8080/9070, SurrealDB `SURREALDB_ENDPOINT` (8000), web 5173,
   Surrealist 18188 — e.g. `lsof -nP -ti:8787,9080,8080,9070,8000,5173 | xargs -r kill`,
   then `pkill -f 'just dev'` / `pkill -f 'brainial_lain_backend'` for strays.
   (`nc -z` can briefly report a port "occupied" on a lingering TIME_WAIT socket;
   re-check before killing.) `just dev` clears `.restate` and starts the stack;
   wait for backend health at `http://127.0.0.1:8787`, then run the gates:
   `just e2e-workflows`, `just e2e-chain`, `just e2e-schedules`. Skip
   `e2e-chat-run` (nondeterministic, not a gate). Tear down when done.
5. **Commit & report.** All gates green → commit `Upgrade backend to upstream
   lash <tag>` (rev bump + app edits). Any gate red → don't commit, leave staged,
   report the failure. Either way, write a `/picturethis` report as an HTML file
   at the repo root (next to the prior `onboarding-*.html` / `lashlang-*.html`)
   covering the changelog, the new lash capabilities that mattered, the app
   changes, and the verification result.

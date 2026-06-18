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

1. **Survey** (read-only). `git -C ../lash pull`. Find the current pinned rev,
   pick the target tag (`git -C ../lash tag --sort=-creatordate | head -1`), read
   `git -C ../lash log/diff <current>..<target>`, and grep `app/` for impact.
   Classify: **none** / **mechanical** / **risky**.
2. **Gate.** none or mechanical → proceed. risky → pause, show the summary + the
   exact edits you propose, wait for go-ahead.
3. **Apply.** Replace the rev SHA on every lash line (`replace_all`). Then in
   `app/backend`: `cargo update` → make app edits → `cargo check`. `cargo check`
   is a hard fail-gate; don't move on while it's red.
4. **Verify.** `just dev` in the background (clears `.restate`, starts the stack;
   wait for backend health at `http://127.0.0.1:8787`), then run the gates:
   `just e2e-workflows`, `just e2e-chain`, `just e2e-schedules`. Skip
   `e2e-chat-run` (nondeterministic, not a gate). Tear down when done.
5. **Commit & report.** All gates green → commit `Upgrade backend to upstream
   lash <tag>` (rev bump + app edits). Any gate red → don't commit, leave staged,
   report the failure. Either way, write a `/picturethis` report as an HTML file
   at the repo root (next to the prior `onboarding-*.html` / `lashlang-*.html`)
   covering the changelog, the new lash capabilities that mattered, the app
   changes, and the verification result.

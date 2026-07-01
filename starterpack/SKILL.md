---
name: starterpack
description: "Deploy a small private web app using the preferred starter architecture: Rust/Axum backend, SolidJS frontend with Zaidan/Kobalte-style design tokens, WorkOS AuthKit, SQLite, LiteFS, and Fly.io. Use when starting, planning, configuring, or deploying this stack; when choosing app defaults; or when the user asks for the Fly/SQLite/LiteFS/WorkOS/SolidJS deployment order."
---

# Starterpack

Use this skill to build and deploy the default private-app starter stack:

- Rust/Axum backend serving both API routes and the built frontend.
- SolidJS frontend, using Zaidan/Kobalte-style primitives and design tokens when a component system is needed.
- WorkOS AuthKit hosted login for identity.
- App-owned session cookies after the WorkOS callback.
- SQLite for application data.
- LiteFS on Fly.io for SQLite durability and primary election.
- One Fly app with one always-on machine in one primary region until there is a clear scaling need.

## Architecture

Prefer a single deployable Rust service:

```text
browser
  -> SolidJS app
  -> Rust/Axum API + auth routes
  -> SQLite database at /litefs/<app>.db
  -> LiteFS FUSE mount
  -> Fly volume + Consul lease
```

Keep the trust boundary on the server. The frontend should call app API endpoints; it should not own OAuth tokens, database access, or privileged WorkOS operations.

## Default Decisions

- Choose SolidJS over Datastar when the app has calendar state, settings, import flows, dashboards, or rich interaction.
- Use Datastar only for very server-shaped CRUD apps where minimal client state is a feature.
- Use WorkOS for identity, not hand-rolled passwords.
- Store private app data in SQLite tables keyed to the app user, not to email strings alone.
- Add an app-level allowlist for private launches even when WorkOS is configured correctly.
- Keep all user data private by default. Make public profile/data sharing explicit and opt-in.
- Start with one machine, one region, one LiteFS volume. Add multi-region only after the write model requires it.

## WorkOS Setup

Use AuthKit hosted UI. Keep these concepts distinct:

- WorkOS Project: environment boundary, such as staging or production.
- WorkOS Application: OAuth/AuthKit client inside a project.
- App redirect URI: the callback route your Rust app implements.
- App sign-in endpoint: the Rust route that starts login.

Production defaults:

```text
Sign-in endpoint: https://<domain>/auth/login
Redirect URI:     https://<domain>/auth/callback
Logout return:    https://<domain>/
```

Required server environment:

```bash
WORKOS_CLIENT_ID=client_...
WORKOS_API_KEY=sk_...
WORKOS_REDIRECT_URI=https://<domain>/auth/callback
ALLOWED_EMAILS=you@example.com
```

Security defaults:

- Enable only intended auth methods in WorkOS.
- Avoid social login unless the product explicitly needs it.
- Reject unverified WorkOS emails.
- Create an app session after callback and store only the minimum identity fields needed.
- Clear the app session on logout before redirecting.
- Never commit WorkOS secrets or local `.env` files.

## Fly And LiteFS Setup

Create the Fly app and LiteFS volume first:

```bash
fly apps create <app-name>
fly volumes create litefs --app <app-name> --region <primary-region> --size 1
fly consul attach --app <app-name>
```

Do not remove `FLY_CONSUL_URL`; LiteFS uses it for the Consul lease.

Use this shape:

```text
Fly volume mount: /var/lib/litefs
LiteFS FUSE dir:  /litefs
SQLite DB path:   /litefs/<app-name>.db
Rust env:         APP_DB_PATH=/litefs/<app-name>.db
```

Start with this Fly machine policy:

```toml
auto_stop_machines = "off"
auto_start_machines = false
min_machines_running = 1
max_machines_running = 1
```

Use a LiteFS config with:

- `fuse.dir` set to `/litefs`
- `data.dir` set to `/var/lib/litefs`
- Consul lease type
- `candidate` true only in the primary region
- `exec` running the Rust binary

## Deployment Order

Follow this order:

1. Buy or choose the domain.
2. Create the WorkOS project and application.
3. Configure the WorkOS redirect URI and sign-in endpoint.
4. Create the Fly app.
5. Create the LiteFS volume in the primary region.
6. Run `fly consul attach --app <app-name>`.
7. Set Fly secrets for WorkOS and app privacy controls.
8. Build the SolidJS frontend.
9. Build the Rust service image.
10. Deploy with `fly deploy --app <app-name>`.
11. Point DNS at Fly.
12. Create/check Fly certificates for apex and `www` if both are used.
13. Test login, callback, session persistence, database writes, and logout.

Set secrets like this:

```bash
fly secrets set --app <app-name> \
  WORKOS_CLIENT_ID='client_...' \
  WORKOS_API_KEY='sk_...' \
  WORKOS_REDIRECT_URI='https://<domain>/auth/callback' \
  ALLOWED_EMAILS='you@example.com'
```

## Verification

Before calling the deployment done, verify:

- `npm run build` passes for the frontend.
- `cargo fmt --check`, `cargo check`, and relevant tests pass for Rust.
- `fly deploy` completes and the machine is healthy.
- `fly logs --app <app-name>` shows the Rust server listening on the expected port.
- `/auth/login` redirects to WorkOS.
- `/auth/callback` returns to the app and creates an app session.
- The app can write to the SQLite database under `/litefs`.
- Logout clears the app session.

## Common Failure Modes

- If the app loops after login, compare `WORKOS_REDIRECT_URI` with the exact WorkOS dashboard redirect URI.
- If LiteFS does not start, confirm `fly consul attach` ran and `FLY_CONSUL_URL` exists.
- If SQLite writes fail, confirm the app uses `/litefs/<app>.db`, not the raw volume directory.
- If the frontend shows stale UI after deploy, hard refresh or check asset caching.
- If production starts without privacy controls, fail startup rather than accepting missing `ALLOWED_EMAILS`.

# Tofu gallery copy

This repository is Tofu's copy of **airisorg/tofu-chat** (licence: MIT), listed in Tofu's app gallery as **Team chat** (团队聊天): Tofu Chat, a team messenger for groups of up to 30 people, with direct messages, group chats, spaces, threads, reactions, search, stars, pictures, files and voice messages.

- `main` mirrors the upstream repository and is never committed to by Tofu.
- `tofu` carries Tofu's own changes; tags named `pin/<date>` mark the versions the gallery points to. The current pin is `pin/2026-10-26.1`.
- A visitor who chooses this app gets a private snapshot of the pinned version in their own Tofu account. Their copy does not follow upstream updates.
- Upstream: https://github.com/airisorg/tofu-chat at `802034c50cad691428d334d182a124ebffee2ce1` (2026-10-25, "Take the sign-in address and database CA from Tofu, not the code (#19)"). The `tofu` branch starts from that commit.

## What the app is

A Next.js 16 app (React 19, TypeScript). Its server keeps every conversation in PostgreSQL through the `postgres` driver (`prepare: false`), in a private schema named `relay`. Identity comes from the copy's own Supabase auth service: every chat, upload and attachment request sends the browser's session as a bearer token, the server verifies it with supabase-js `getUser` and refuses an account whose email is not confirmed. Google sign-in starts at Tofu's sign-in broker. The browser asks for new messages every 3 seconds while the page is visible, and a private realtime event on `relay.events` makes it ask sooner. The landing page's "Explore demo" uses synthetic data in the browser; production builds turn it off unless `NEXT_PUBLIC_ENABLE_DEMO` is set to `true`.

## How Tofu runs it

- Framework preset `nextjs`, root folder: the repository root (the root folder setting is empty). Install: `npm ci` from `package-lock.json`. Build: `npm run build`. There is no output folder setting: the host takes Next.js's own build output.
- `npm run build` is `tsx scripts/verify-database.ts && tsx scripts/tofu-schema.ts && next build`:
  1. `scripts/verify-database.ts` (upstream) opens one TLS-verified connection to `DATABASE_URL`, runs `select 1`, and closes it. A build that cannot connect stops, and so does a hosted build (`VERCEL=1`, or `VERCEL_ENV` `production` or `preview`) without `DATABASE_URL`; a local build without it skips the step.
  2. `scripts/tofu-schema.ts` (Tofu's) applies the app's schema through one connection with the same TLS rules and prints `Chat schema ready: version 6.` See [The relay schema](#the-relay-schema).
  3. `next build`.
- Every step runs through `tsx` or `next` from `node_modules`, so no repository file has to be executable (Tofu deploys files without their Unix mode; this repository has no executable file).
- The settings Tofu writes on the app with its managed database, and the code that reads each:

| Setting                                                     | Read by                                                                                                                                                                                                                                |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DATABASE_URL`                                              | `database()` in `src/lib/server.ts` (the server's pool: at most 3 connections, `prepare: false`); `publicConfig()` there, for `/api/config`'s `databaseConfigured`; `scripts/verify-database.ts`; `scripts/tofu-schema.ts`             |
| `SUPABASE_CA_CERT`                                          | `src/lib/database-tls.ts`, for the server and both build steps: the database's root is trusted beside Node's own roots, and in production peer and host name are always verified. A `DATABASE_CA_CERT` the owner sets replaces it      |
| `SUPABASE_URL`, `SUPABASE_ANON_KEY`                         | `publicConfig()` and `authenticatedUser()` in `src/lib/server.ts` (the `getUser` check; the address and the public key reach the browser through `/api/config`); `src/proxy.ts` puts the address in the page's Content Security Policy |
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` | The same code, when the names without the prefix are empty                                                                                                                                                                             |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`                      | Nothing in this app                                                                                                                                                                                                                    |
| `TOFU_SIGN_IN_URL`, `NEXT_PUBLIC_TOFU_SIGN_IN_URL`          | `publicConfig()`: where the Google button starts, passed on only as a plain `https` address. Without it the Google button stays off                                                                                                    |

- `DATABASE_URL` is the managed database's transaction pooler (Supavisor in transaction mode, port 6543, `?sslmode=require`). The app's explicit TLS options outrank the URL's `sslmode`.
- At the copy's first deployment Tofu also writes Google sign-in into the copy's own auth service; email sign-ups there must be confirmed by email.

## The relay schema

`SCHEMA` and `applySchema` in `src/lib/server.ts` are upstream's. `scripts/tofu-schema.ts` imports `applySchema` and runs it; it copies no SQL.

- Schema `relay`, 12 tables: `schema_migrations`, `profiles`, `conversations`, `participants`, `invites`, `messages`, `reactions`, `stars`, `events`, `uploads`, `operations`, `request_limits`, with 21 indexes (12 primary keys, the unique `profiles.email` and 8 more). Nothing else: no view, sequence or function.
- Row-level security is on for all 12 tables. The server connects as the role that created the tables, their owner, which row-level security does not restrict, and checks membership and authorship itself in its transactions.
- Everything on the schema and its tables is revoked from `PUBLIC`, `anon` and `authenticated`. Then `authenticated` gets `USAGE` on `relay` and `SELECT` on `relay.events` only, with the policy `relay_events_own_select` (`FOR SELECT TO authenticated USING (user_id = auth.uid()::text)`): a signed-in browser can read its own event rows (an id, its own user id, a conversation id and a time; no message text), which is what the realtime nudge needs.
- `relay.events` is added to the `supabase_realtime` publication, and version 6 is recorded in `relay.schema_migrations`.
- When: at every build (Tofu's step), and, as upstream does, at each server process's first authenticated request. Once version 6 is recorded, both return right after taking the lock.

### When several run at once

`applySchema` does all of its work in one transaction whose first statement is `select pg_advisory_xact_lock(724931108)`. The one holding the lock checks for version 6 and either returns or runs the whole schema; the lock ends with that transaction's `COMMIT`. Every other caller (another server instance, the build of a new version while the old one serves, a second build) waits for the lock, then finds version 6 and returns. Inside one server process a single pending promise (`migration`) makes concurrent requests share one run.

This works through Supavisor's transaction mode because it uses nothing outside the transaction: the lock is a transaction lock, taken first, inside the one transaction that also creates the schema and records the version; the pooler keeps one database connection for the whole transaction, so the lock, the schema and the record go through one backend, and the lock is released by the same `COMMIT` or `ROLLBACK`. No session-level lock, `SET` or prepared statement is involved (`prepare: false`).

## Sign-in

- Google through Tofu's broker (`TOFU_SIGN_IN_URL`); the session is the copy's own auth service's. As upstream has it, the sign-in callback must match a sign-in request that the same tab saved within the last ten minutes.
- Anyone with a Google account can sign in to a new copy until its owner chooses "Only people you add" under "Who can sign up" on the app's Sign-in page in Tofu's dashboard. Once that is chosen, someone who has never signed in gets in only after the owner adds their Google address on that same page: adding them to a conversation in the app is not enough. The app has no sign-up setting of its own and no administrator: every account is an ordinary member.
- A stranger who signs in gets an empty workspace. They see only conversations they belong to, or are invited to by their verified email; the server builds each person's list of people only from those conversations, so there is no directory of the copy's people. They cannot read anyone else's messages, files or conversations. They can start conversations of their own and send files, which use the copy's database, within upstream's limits (30 people per conversation; per account and minute, 600 reads and 120 writes, upload chunks or downloads).

## The database's Data API

Supabase's Data API (PostgREST) serves only the schemas set as exposed, by default `public` and `graphql_public`; `relay` is not one of them, so the Data API does not serve it. Even if somebody exposed `relay`, `anon` has no privilege on the schema or any table in it (`permission denied for schema relay`), and `authenticated` can only read its own `relay.events` rows through the policy above and write nothing. The app keeps nothing in `public`.

## What works differently on Tofu

- No email is sent. An invitation is a link that people share themselves; the invited person joins by signing in with the invited email (upstream sends no email either).
- Pictures, files and voice messages are stored in the database (up to 5 MiB each, three per message; voice up to two minutes) and count toward the database's size. Uploads go in 1 MiB chunks, in requests below the host's request limit.
- No video or voice calls, no background push notifications and no automatic sending while offline, as upstream says.
- The demo stays off in production unless `NEXT_PUBLIC_ENABLE_DEMO` is set to `true`.

## What Tofu changes

Only these, each in its own commit on `tofu`:

1. `scripts/tofu-schema.ts`, its place in `package.json`'s build script between `verify-database` and `next build`, and two tests (`tests/tofu-schema.test.ts`, `tests/tofu-schema-pglite.test.ts`). Why: without it a new copy's database has no tables until somebody signs in, because the server applies its schema only at the first authenticated request. With it the database has its 12 closed tables right after the first deployment, so the owner's Database page shows them, the first sign-in does not wait for the schema, and Tofu's live check can verify the tables and their access rules on a real Tofu database, which it cannot otherwise do (no agent can complete a Google sign-in). The step opens one connection with exactly `verify-database`'s TLS rules (production verification, `SUPABASE_CA_CERT` or `DATABASE_CA_CERT`; `prepare: false`, one connection; connect within 15 s, schema within 60 s, close within 3 s), prints one line naming the version and never a host, user, database name or credential, skips a local build without `DATABASE_URL`, refuses a hosted one, and stops the build with `Chat schema setup failed. Deployment stopped; check the database connection settings.` instead of the driver's text. The runtime path is exactly upstream's.
2. `README.md`: the sentence saying this server app with a database needs Tofu's Pro hosting was out of date (Tofu's Free plan can host one server app with its database). It now says only that it is a server app with a database and keeps the link to the current plans.
3. This file.

Chat behaviour, the interface, the demo, the security model and the dependencies (`package-lock.json`) are upstream's.

## Needs a real deploy to verify

- A real Google sign-in through Tofu's broker and the copy's auth service.
- Realtime through the copy's auth and realtime services: the private `postgres_changes` subscription on `relay.events` with the person's token.
- Uploads through the host's request limit.
- Both build steps through Tofu's real transaction pooler and its certificate (here a local PostgreSQL 14 with a throwaway CA, reached directly), and Tofu's live check reading the 12 tables and their access rules after the first deployment.

## Licence

- MIT, "Copyright (c) 2026 Chat contributors". `LICENSE` is upstream's, unchanged. The bundled fonts, icons and emoji data, and the Google sign-in mark, keep their own licences and terms, listed in `THIRD_PARTY_NOTICES.md`. The npm packages in `package-lock.json` keep their own licences.
- Every new file says at its top that it is Tofu's addition. The upstream files Tofu changed are `package.json` (the build script) and `README.md` (one sentence).

## Verification

### `pin/2026-10-26.1`

This file's wording only (the demo setting must be `true`; which requests carry the session; what "Only people you add" asks of the owner). The code is `pin/2026-10-26`'s, verified as below.

### `pin/2026-10-26`

2026-10-26, on macOS arm64 against 127.0.0.1 only. No Docker, nothing deployed. The code is the `tofu` branch at `16859f7` (both code commits); the commit after it adds only this file.

- In a clone, after `npm ci`: `npm run typecheck` and `npm run lint` passed; `npm run test:unit:fast` 22 files, 172 tests passed (7 of them new); `npm run test:sql` 14 files, 76 tests passed (1 new); `npm run test:coverage` passed (all non-native Node tests with upstream's coverage floors); `npx prettier --check .` passed. `npm run test:native` (after a local production build and its source binding, as `docs/testing.md` describes) with Homebrew PostgreSQL 14.24 and OpenSSL 3.6: 3 files, 16 tests passed; `docs/testing.md` asks for PostgreSQL 15 or newer, and none was installed. Upstream's browser suites were not run: this version changes nothing they reach.
- The schema race, on a scratch PostgreSQL 14.24 cluster on 127.0.0.1 with TLS (a throwaway CA and a server certificate for `127.0.0.1` and `localhost`), made to look like Supabase: `anon` and `authenticated` roles (`NOLOGIN`), a schema `auth` with `auth.uid()` returning `uuid` from `request.jwt.claim.sub`, an empty `supabase_realtime` publication, and the app connecting as an ordinary login role that owns the database (not a superuser); `log_statement = all`, `log_lock_waits = on`, `deadlock_timeout = 20ms`. Each case on a new, empty database:
  - 5 separate Node processes, each with its own verified TLS connection, released at the same instant into `applySchema`: 5 of 5 finished without an error; 5 lock requests; the schema's statements ran in 1 backend; 4 waits on the lock were logged, each followed by the lock; each call took 29 to 34 ms; no `ERROR` or `FATAL` line.
  - 10 processes, the same way: 10 of 10 without an error; 10 lock requests; the schema ran once; 8 waits longer than 20 ms logged; 21 to 32 ms each; no `ERROR` or `FATAL` line.
  - The build's step itself (`tsx scripts/tofu-schema.ts`, as the build script runs it, with `DATABASE_URL` and `SUPABASE_CA_CERT`) and 4 processes, all 5 queued on a held advisory lock (5 waiters seen in `pg_locks`) and let go at once: 5 of 5 exited 0; the build step ran the schema and printed `Chat schema ready: version 6.`; the 4 others waited and returned; no `ERROR` or `FATAL` line.
  - Two concurrent calls inside one Node process through `enforceRequestLimit`, the exported function that reaches the internal `database()` helper without a Supabase session (it reads only the user's id): 1 lock request, the schema ran once, both calls succeeded.
  - Read back from the catalogs after each case: `relay` holds 12 tables, all 12 with `relrowsecurity` (none forced), and 21 indexes, nothing else; `anon` has no privilege on `relay` or any table, column or function in it; `authenticated` has `USAGE` on `relay` and `SELECT` on `relay.events` and nothing else there; `PUBLIC` has nothing; `relay.events` is in `supabase_realtime` once and is its only table; `relay.schema_migrations` holds version 6 once; `relay_events_own_select` exists once (`SELECT`, `{authenticated}`, `(user_id = (auth.uid())::text)`). As `anon`, reading `relay.events` or `relay.profiles` failed with `permission denied for schema relay`; as `authenticated`, reading `relay.events` returned 0 rows, reading `relay.messages` failed with `permission denied for table messages`, and inserting into `relay.events` failed with `permission denied for table events`.
- The build's step with a CA other than the database's, and with no CA (Node's default store): both stopped with the sanitized line and exit code 1, and `relay` was not created.
- Built from a `git archive` of `16859f7` with every execute bit removed (764 files, 0 executable before or after): `npm ci`, then `npm run build` with `DATABASE_URL` (`?sslmode=require`) to a new database of the TLS cluster above and its CA as `SUPABASE_CA_CERT`: `Database TLS verification passed.`, `Chat schema ready: version 6.`, then Next.js 16.4.3 built `/`, `/api/attachments`, `/api/chat`, `/api/config` and `/api/uploads`. `next start` on 127.0.0.1 answered `/` 200, `/api/config` 200 with `"databaseConfigured":true`, and `/api/chat` 401 without a session (`{"error":"Sign in to continue."}`). The catalogs read back as above. The clone's own build for the native tests, without `DATABASE_URL`, printed `Database TLS verification skipped: no local DATABASE_URL.` and `Chat schema step skipped: no local DATABASE_URL.`
- `src/lib/server.ts` at build time: its imports are supabase-js, `postgres`, `node:crypto` and the app's own `database-tls`, `action-identity`, `chat-limits`, `native-emoji`, `media-limits` and types; at module scope it only defines constants, classes and functions, and it reads every setting inside a function. Importing it needs no request and no browser; the build step ran it in a plain Node process before `next build`.
- Tofu's own check on `tofu` at `16859f7`: `detect=nextjs files= 764 published= 736 -> not on the static lane | refused: configurationx47, typex155, server_codex4 | build: not_vite | safety ok (0 findings)`, `archive 200 10779677`.
- Not run here: Tofu's transaction pooler, Supabase's Data API, Auth and Realtime services, Google sign-in, Vercel's runtime and request limit, and anything on Tofu.
- Every local server and the scratch database cluster were stopped, and the build output was deleted afterwards.

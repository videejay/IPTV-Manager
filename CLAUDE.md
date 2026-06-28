# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install          # Install dependencies (use npm, not pnpm — Docker/CI use package-lock.json)
npm start            # Run the server (src/server.js)
npm run lint         # ESLint over src/
npm test             # Run all Vitest tests (no file parallelism)
npm run test:watch   # Run Vitest in watch mode

# Run a single test file
npm exec vitest run tests/auth_middleware.test.js

# Isolated test run (prevents touching repo-root DBs)
DATA_DIR="$(mktemp -d)" npm test
```

Node.js 24+ is required. `better-sqlite3` is a native module — reinstall with `npm install` after changing Node versions.

## Architecture

### Process Model

`src/server.js` uses **Node.js cluster** to fork one worker per CPU. The first worker is also the **scheduler worker** (`IS_SCHEDULER=true`), running sync, EPG, cleanup, SSDP, and GeoIP update jobs. All workers share two SQLite databases via WAL mode. An optional Redis (`REDIS_URL`) can replace the SQLite-backed stream tracker for multi-instance deployments.

### Application Entry Points

- `src/server.js` — cluster bootstrap, DB init, Redis setup, scheduler startup
- `src/app.js` — Express app: middleware stack, all route registrations

### Data Layer

Two SQLite databases under `DATA_DIR` (defaults to repo root; Docker sets `/data`):

- **`db.sqlite`** (`src/database/db.js`) — main app data: users, providers, channels, categories, EPG sources/mappings, sync configs, stream sessions, settings, security logs
- **`epg.db`** (`src/database/epgDb.js`) — EPG channel and programme data, keyed by `(source_type, source_id)`

Migrations run in `src/database/migrations.js` via `initDb(true)` in the primary process before workers start. Migrations must be idempotent and guarded by schema checks or `settings` marker rows.

Provider credentials (passwords) are encrypted at rest using `src/utils/crypto.js`.

### Request Flow

1. **Security middleware** (`src/middleware/security.js`) — Helmet headers, IP blocking, rate limiters (API/auth/client-log, all tunable via env vars)
2. **Auth middleware** (`src/middleware/auth.js`) — JWT verification (`HS256`), checks `token_version` for revocation, enforces GeoIP region lock and WebUI access flag
3. **Routes** (`src/routes/`) → **Controllers** (`src/controllers/`) → **Services** (`src/services/`)

### Route Groups

All admin/user API routes are prefixed `/api`. Xtream Codes–compatible endpoints live at root paths:

| Prefix | Purpose |
|---|---|
| `/api` | Auth, users, providers, channels, EPG, system, shares, backups |
| `/player_api.php` | Xtream Codes metadata and category/stream listings |
| `/live/:user/:pass/:id.ts` | Live stream proxy |
| `/movie/:user/:pass/:id.ext` | VOD proxy |
| `/series/:user/:pass/:id.ext` | Series proxy |
| `/xmltv.php` | XMLTV EPG output (gzip when accepted) |
| `/hdhr/:token/...` | HDHomeRun device emulation |
| `/share/:slug` | Public short-link redirect |

### Key Services

- **`streamManager`** (`src/services/streamManager.js`) — tracks active streams per worker; uses Redis hash when available, falls back to `current_streams` SQLite table. Enforces per-user and per-provider connection limits.
- **`syncService`** (`src/services/syncService.js`) — pulls channels and categories from IPTV providers via Xtream API or M3U; updates `provider_channels` and `category_mappings`.
- **`epgService`** (`src/services/epgService.js`) — fetches and parses XMLTV (plain or gzip) into `epg.db`. Self-signed HTTPS is allowed for EPG sources only.
- **`channelMatcher`** (`src/services/channelMatcher.js`) — fuzzy channel-to-EPG matching using bitwise signatures, language prefix stripping, and ISO 639 normalization. Run in a **worker thread** (`src/workers/epgWorker.js`) for CPU isolation.
- **`schedulerService`** (`src/services/schedulerService.js`) — cron-style scheduler for provider sync, EPG updates, stream cleanup, and GeoIP updates.
- **`cacheService`** (`src/services/cacheService.js`) — in-process `Map` cache for generated channel JSON payloads, keyed per user. Invalidated on channel/category mutations.
- **`geoIpService`** / **`geoIpUpdateService`** — region locking for users; MaxMind GeoLite2 update, checksum-gated to avoid redundant downloads.

### Network Safety

All outbound HTTP calls must go through **`fetchSafe`** (`src/utils/network.js`), which enforces SSRF protection via `isSafeUrl` / `safeLookup` (blocks private IPs, localhost, cloud metadata). Custom DNS lookup agents are also used directly in `streamController.js` for stream proxying.

### Frontend

Static files served from `public/`. Bootstrap 5 + vanilla JS — no build step. Internationalization (EN/DE/FR/EL) is handled client-side.

## Working Rules

- Read the relevant controller, route, service, and DB schema before editing.
- Keep API responses backward-compatible unless explicitly requested.
- Avoid broad refactors in controllers that handle streaming, auth, provider sync, EPG, imports/exports, or database migrations.
- When changing routes, environment variables, setup, Docker behavior, or integration behavior, update `README.md`, `docs/API_REFERENCE.md`, `docs/CONFIGURATION.md`, or `docs/DEVELOPMENT.md` as relevant.
- Do not introduce new dependencies without clear justification.
- All outbound HTTP calls must use `fetchSafe`; never use bare `fetch` or `node-fetch` for external URLs.
- New migrations must be idempotent and must preserve existing user/provider/channel IDs.

## Runtime Data (Never Commit)

`db.sqlite*`, `epg.db*`, `secret.key`, `jwt.secret`, `cache/`, `temp_*`, `temp_uploads/`

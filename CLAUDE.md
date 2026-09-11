# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

AmneziaWG Easy (`awg-easy`) is a fork of `wg-easy` that bundles the AmneziaWG userspace daemon (obfuscated WireGuard) with a Node.js backend and a Vue 2 (CDN, no build step) web UI for managing VPN clients. Everything ships as a single Docker image; the app has no test suite, no bundler for the frontend JS, and no database — all state lives in a JSON file (`wg0.json`) plus the generated `wg0.conf` on disk.

## Commands

Root `package.json` scripts wrap Docker:
```bash
npm run build       # docker build --tag wg-easy .
npm run serve       # docker compose up (uses docker-compose.yml + docker-compose.dev.yml)
npm run start       # docker run ... (production-like manual run)
```

Inside `src/` (the actual Node app):
```bash
cd src
npm run serve                # nodemon server.js with DEBUG=Server,WireGuard (runs outside Docker, needs wg/awg CLI on PATH)
npm run serve-with-password  # same, with PASSWORD=wg
npm run lint                 # eslint . (eslint-config-athom, see src/.eslintrc.json)
npm run buildcss              # rebuild www/css/app.css from www/src/css/app.css via Tailwind
```

There are no automated tests in this repo. Verify changes by running the app (Docker or `npm run serve` on a Linux host with WireGuard/AmneziaWG kernel support) and exercising the Web UI / API by hand.

## Architecture

**Backend** (`src/server.js` → `src/services/*` → `src/lib/*`):
- `src/services/Server.js` / `src/services/WireGuard.js` are thin singletons (`module.exports = new X()`) wrapping the real classes in `src/lib/Server.js` and `src/lib/WireGuard.js`. Always import via `services/*`, not `lib/*` directly, to keep the singleton behavior.
- `src/lib/WireGuard.js` is the core: it owns the on-disk config (`wg0.json` = source of truth, JSON) and regenerates `wg0.conf` (the real `wg-quick`/`awg-quick` config) on every mutation. Client/server CRUD methods (`createClient`, `deleteClient`, `enableClient`, `updateClientAddress`, etc.) all end by calling `saveConfig()`, which writes `wg0.json` + `wg0.conf` and then does `wg syncconf wg0 <(wg-quick strip wg0)` to apply changes live without restarting the interface. Interface bring-up/down uses `wg-quick up/down wg0`.
- Server config uses `Address = <server>/24`; client configs use `Address = <client>/32` (peer `AllowedIPs` on the server side is `/32` per client). Don't reintroduce `/24` on the client template — that was a deliberate fix (routes the whole subnet on-link on some client platforms and breaks tunneled traffic).
- AmneziaWG obfuscation parameters (`Jc/Jmin/Jmax`, `S1/S2`, `H1-H4`) are generated once per server (random defaults, overridable via env — see `src/config.js`) and stored in `wg0.json.server`; they're written identically into both the server `[Interface]` block and every client config, since obfuscation must match on both ends.
- `src/lib/Server.js` is the HTTP layer, built on `h3` (not Express) with hand-rolled routers (`createRouter()` instances chained per concern: session/auth, wireguard client CRUD, prometheus metrics, backup/restore, static file serving). Auth is session-cookie (`express-session` via `fromNodeMiddleware`) or an `Authorization` header carrying the raw password, checked with bcrypt against `PASSWORD_HASH`. Route params like `clientId` are explicitly checked against `__proto__`/`constructor`/`prototype` to block prototype-pollution via routing — preserve that guard on any new `:clientId` route.
- One-time client-config download links (`/cnf/:clientOneTimeLink`) and client expiry are optional features gated by `WG_ENABLE_ONE_TIME_LINKS` / `WG_ENABLE_EXPIRES_TIME`, ticked by a self-rescheduling `setTimeout` cron loop (`cronJobEveryMinute` in both `Server.js` and `WireGuard.js`) — no external scheduler.
- All runtime configuration is centralized in `src/config.js`, reading from `process.env` with defaults; it's the single place to add a new environment variable. `README.md`'s Options table documents every env var and must stay in sync when config.js changes.

**Frontend** (`src/www/`): static Vue 2 app loaded via `<script>` tags from `js/vendor/` (Vue, vue-i18n, ApexCharts, sha256, timeago) — no npm/webpack build for JS. `js/api.js` is the fetch wrapper for the backend REST API, `js/app.js` is the Vue root component/app logic, `js/i18n.js` holds translation strings for all supported languages. Only the CSS goes through a build step (Tailwind, `npm run buildcss`).

**Docker image** (`Dockerfile`): multi-stage build — stage 1 builds `node_modules` on `node:24-alpine`; stage 2 is `FROM amneziavpn/amneziawg-go:<pinned tag>` (Alpine-based, provides `awg`/`awg-quick` binaries symlinked to `wg`/`wg-quick`, plus the `amneziawg-go` userspace daemon), onto which the app and `node_modules` are copied and `nodejs`/`npm`/`iptables` are installed via `apk`. `iptables-legacy` is forced (symlinked over `iptables`) for compatibility. The base image tag is pinned deliberately (not `latest`) for reproducible builds — bump it consciously and re-test handshake/traffic, since it also carries the AmneziaWG protocol tooling version.

**CI** (`.github/workflows/deploy.yml`): on push to `master` (repo `YokiToki/awg-easy` only), builds and pushes a `linux/amd64`-only Docker image to `ghcr.io/yokitoki/awg-easy`, tagged `latest` and with the version from `src/package.json`'s `release.version` field (this should track the base `amneziawg-go` image tag, per README's changelog).

### Keep in sync when bumping the `amneziawg-go` base image

Whenever the base image tag in `Dockerfile` (`FROM amneziavpn/amneziawg-go:...`) changes, also update:
- `src/package.json`'s `release.version` field to match the new tag — this is what CI (`deploy.yml`) uses to tag the published image and what the Web UI shows via `/api/release`. It is a deliberate project convention (see README's "Changes by Stanislav Karakovskii" changelog: "Release number now matches amneziawg-go version"), not incidental.
- Optionally, README.md's changelog section at the bottom, if you want to keep a human-readable history of base-image bumps.
- Re-verify manually afterwards (no automated tests cover this): rebuild the image, bring up a client, confirm handshake and actual traffic — a base-image bump can change `amneziawg-tools` (`awg`/`awg-quick`) protocol behavior even when the `amneziawg-go` daemon version string doesn't change.

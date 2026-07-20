# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`simple-proxy` is a CORS-bypass reverse proxy used by the [movie-web](https://movie-web.app)/[P-Stream](https://pstream.org) media-streaming clients (see the sibling `pstream` repo). It is built on **Nitro** (h3 under the hood) and deploys to multiple targets from one codebase. Docs: https://docs.pstream.org/proxy/introduction.

It exposes a stable public URL contract consumed by external clients — `?destination=`, `/m3u8-proxy?url=&headers=`, `/ts-proxy?url=&headers=` — so route signatures should stay backward compatible unless coordinating a change with consumers.

## Commands

- `pnpm install` — this repo is **pnpm-only**; `preinstall` runs `npx only-allow pnpm` and blocks npm/yarn.
- `pnpm dev` — local dev server (`nitropack dev`).
- `pnpm build` — default (Node) build.
- `pnpm build:cloudflare` / `build:aws` / `build:node` / `build:netlify` — preset-specific builds (set `NITRO_PRESET` and rerun `nitropack build`).
- `pnpm start` — run the built Node output (`node .output/server/index.mjs`).
- `pnpm lint` / `pnpm lint:fix` — ESLint over `src/` (`.ts` only).
- No test suite exists in this repo.

## Architecture

Routes are **file-based** under `src/routes/` (Nitro convention — filename determines the route path, no explicit router):

- `routes/index.ts` — main proxy route (`/`). Handles CORS preflight, the Turnstile bot-protection gate, and generic reverse-proxying via h3's `sendProxy`/`specificProxyRequest`.
- `routes/m3u8-proxy.ts` — `/m3u8-proxy`: fetches and rewrites HLS (`.m3u8`) playlists so every segment/key/variant URL routes back through this proxy (defeats IP-based CDN blocking). Also serves `/cache-stats`. Runs an in-memory segment prefetch/cache.
- `routes/ts-proxy.ts` — `/ts-proxy`: serves individual `.ts` segments (or keys), preferring the in-memory cache populated by `m3u8-proxy.ts`.
- `utils/headers.ts` — maps `X-`-prefixed override headers (e.g. `X-Cookie` → `Cookie`) so clients can set protected headers the browser normally blocks; also strips CF/AWS/forwarded headers before the upstream fetch and sets response CORS headers.
- `utils/proxy.ts` — `specificProxyRequest()`, the shared generic-proxy core (header merge/filter, streamed or buffered request bodies).
- `utils/turnstile.ts` — Cloudflare Turnstile verification + short-lived JWT issuance/verification gating the main route.
- `utils/ip.ts` — `getIp()` reads `CF-Connecting-IP`; **only works on Cloudflare Workers**, so Turnstile protection is effectively Cloudflare-only even though the app itself is multi-platform.
- `utils/body.ts` — raw body reading for PUT/POST/PATCH/DELETE.

### Multi-target build system

One Nitro codebase, multiple deploy presets, each governed by a different config file:
- `nitro.config.ts` — shared Nitro config (`srcDir: src`, `@` alias, exposes `package.json` version via `runtimeConfig.version` at the `/` health-check).
- `wrangler.toml` — Cloudflare Workers (paired with `build:cloudflare`).
- `netlify.toml` — Netlify Edge Functions (paired with `build:netlify`; `command = "pnpm build:netlify"`).
- `Dockerfile` — Node 20 Alpine multi-stage build using the default Node preset, for Docker/plain-Node deploys.
- AWS Lambda uses `build:aws` with no separate config file beyond the `NITRO_PRESET` env var.

## Environment variables

No `.env.example` exists — these are only documented in source:

| Variable | Effect |
|---|---|
| `TURNSTILE_SECRET` | Cloudflare Turnstile secret key. Unset (together with `JWT_SECRET`) disables Turnstile protection entirely. |
| `JWT_SECRET` | Signs/verifies the 10-minute JWT issued after Turnstile verification. |
| `DISABLE_CACHE` | `"true"` disables in-memory `.ts` segment caching/prefetching. |
| `DISABLE_M3U8` | `"true"` disables the `/m3u8-proxy` and `/ts-proxy` routes (404). |
| `REQ_DEBUG` | `"true"` logs outgoing proxied request method/url/headers. |

## Conventions

- TypeScript strict mode, ES2020 target, `@/*` → `src/*` path alias.
- ESLint: `@typescript-eslint/recommended` + `prettier`; `no-console`, `no-explicit-any`, `no-await-in-loop`, `no-nested-ternary` are all off; `no-shadow` is an error; unused vars warn with `_`-prefix ignored. Lint scope is `src/` only.
- Prettier: trailing commas, single quotes.
- LF line endings, 2-space indent, final newline (`.editorconfig`).

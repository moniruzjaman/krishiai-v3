# Krishi AI Production Build

Krishi AI is a resilient, offline-first agricultural intelligence platform for Bangladesh. This repository is initialized as a monorepo with strict runtime isolation between the Vercel Node.js runtime and Cloudflare Workers edge runtime.

## Architecture Map

```text
/apps
  /web                  # Vite + React PWA, deployed by Vercel
  /workers              # Cloudflare Workers utility APIs, deployed by Wrangler
/packages
  /database             # Local-first schema contracts and Supabase sync engine
  /shared-ui            # Shared React/Tailwind-ready UI primitives
/api                    # Vercel Serverless Functions for AI and auth orchestration
```

## Runtime Isolation Contract

| Boundary | Runtime | Owns | Secrets | Must Not Do |
| --- | --- | --- | --- | --- |
| `apps/web` | Browser via Vite/React | PWA UI, offline indicators, local database reads, client API calls | `VITE_*` only | Import server secrets or Cloudflare bindings |
| `api` | Vercel Node.js functions | Gemini/Groq orchestration, Supabase auth/session automation | `process.env.*` | Import Cloudflare `env` bindings |
| `apps/workers` | Cloudflare Workers | Weather, market, and soil utility APIs | Worker `env.BINDING` values | Read `process.env` or import Node-only modules |
| `packages/database` | Browser-compatible package | WatermelonDB schema plan, minimum-info records, online-only sync adapter contract | None | Make direct server-side secret calls |
| `packages/shared-ui` | Browser-compatible package | Shared UI primitives | None | Perform data fetching |

## Edge API Responsibilities

Cloudflare Workers own high-frequency agricultural utility reads:

- `GET /weather?lat=...&lon=...&days=7` &mdash; Open-Meteo scaffold for soil moisture, evapotranspiration, and GDD inputs.
- `GET /soil?lat=...&lon=...` &mdash; SoilGrids scaffold for pH and nutrient-related properties.
- `GET /market?crop=rice` &mdash; DAM market-price adapter placeholder kept on the edge runtime.
- `GET /health` &mdash; Worker runtime health check.

The frontend reaches these routes only through `apps/web/src/services/edge-client.ts`, which defaults to `https://cf.krishiai.live` and can be overridden with `VITE_CF_EDGE_URL`.

## Vercel API Responsibilities

Vercel owns AI and auth orchestration:

- `POST /api/chat` &mdash; AI route scaffold for Gemini 2.0 Flash primary responses and Groq fallback in the next implementation phase.
- Existing `/api/analyze` remains available during migration for the current app experience.

The frontend reaches Vercel AI only through `apps/web/src/services/ai-client.ts`.

## Offline-First Data Strategy

The `@krishiai/database` package establishes the local-first contract that will be backed by WatermelonDB:

- UI reads from the local database first.
- Sync runs only when `navigator.onLine` reports network availability.
- The minimum-information policy limits persisted farmer data to anonymized GPS, crop type, and chat-history references.
- Supabase push/pull is abstracted behind a `SyncAdapter` so secrets stay out of the browser bundle.

## Environment Variable Rules

| Target | Prefix / Access Pattern | Examples |
| --- | --- | --- |
| Vite frontend | `import.meta.env.VITE_*` | `VITE_CF_EDGE_URL`, `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` |
| Vercel functions | `process.env.*` | `GEMINI_API_KEY`, `GROQ_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY` |
| Cloudflare Workers | `env.BINDING` | `OPEN_METEO_BASE_URL`, `SOILGRIDS_BASE_URL`, `DAM_MARKET_BASE_URL` |

## Development Commands

```bash
npm install
npm run dev              # Vite web app
npm run build            # Production web build
npm run dev:worker       # Cloudflare Worker local dev
npm run build:worker     # Worker typecheck/build validation
npm run typecheck        # Web + worker type checks
```

## Implementation Phases

1. **Foundation** &mdash; Monorepo directories, Vite/Vercel config, Wrangler config, runtime boundary documentation.
2. **Data Layer** &mdash; WatermelonDB models, migrations, encrypted IndexedDB/SQLite adapter, Supabase sync adapter.
3. **Auth** &mdash; Device-linked anonymous Supabase session mapping and zero-friction auto-login.
4. **API Isolation** &mdash; Production Gemini/Groq AI routes on Vercel and production utility routes on Workers.
5. **UI/UX** &mdash; Offline-ready dashboard, agricultural charts, sync state, and local cache recovery flows.

## Validation Rules

- No `process.env` usage in `apps/workers`.
- No Cloudflare Worker `env` binding usage in `api`.
- Browser services are split between Vercel AI (`ai-client.ts`) and Cloudflare edge utilities (`edge-client.ts`).
- The dashboard must remain functional from cached/local data in airplane mode as the data layer is completed.

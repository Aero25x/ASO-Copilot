<img width="1920" height="995" alt="image" src="https://github.com/user-attachments/assets/7fbcef3e-d660-491b-affe-815676ec0fbc" />

One price. Every tool.

Even the ones we ship next.

[Persocia.com](https://persocia.com)

---

# ASO Copilot

A self-hosted, free alternative to tools like rankd.dev/AppTweak/ASOMobile for developers managing
their own apps. Single-user, no accounts, no billing — just your apps.

Also exposes an MCP server so an agent (Claude Code, Claude Desktop, etc.) can manage your ASO
directly: add apps, track keywords, pull health reports, spy on competitors, find keyword
opportunities, and write AI copy suggestions itself with no separate API key.



https://github.com/user-attachments/assets/970ce32e-a980-42ce-a0eb-529c425439af




## Features

- **Daily keyword rank tracking** — real ranks from live App Store / Google Play search results
- **Unlimited apps, keywords, competitors** — it's your database, no plan limits
- **Competitor spy** — track competitor apps and see their rank for every keyword you track
- **Health score analysis** — title/subtitle/screenshots/description/ratings/freshness breakdown
  with suggestions, same shape as the paid tools
- **Keyword research** — "find winning keywords" derives candidates from your + competitors'
  metadata and scores them by estimated demand/difficulty using live store search data
- **Global Reach** — scans your rank for a keyword across ~40 major App Store / Google Play
  storefronts, plotted on a world map
- **Review analysis** — rating distribution, positive/negative theme extraction, and a keyword-gap
  finder that mines your own reviews for untracked search terms
- **Product health overlay** — pairs your app's real daily active users (via a per-app PostHog
  project) against its ASO health score over the same window
- **AI Copy Suggestions** — rewrites title/subtitle/description for 8 locales (US/UK/Canada
  English, Spanish, German, French, Portuguese, Japanese), adapting keywords into real local
  search terms rather than translating literally. Two ways to generate it, no lock-in to either:
  - via [OpenRouter](https://openrouter.ai) (any backing model) through the web UI, or
  - **key-free**, by pointing any MCP-connected model (e.g. Claude Code) at the
    `prepare_copy_localization_brief` / `save_copy_suggestions` tools — the model composes the
    copy itself, no API spend beyond what you're already paying for the agent
- **MCP server** at `/api/mcp` for agent-driven ASO management

## Stack

Next.js (App Router) + Prisma + PostgreSQL. Store data comes from the public iTunes Search/Lookup
API (App Store) and `google-play-scraper` (Google Play). There is no free official API for search
volume or keyword rank position, so:

- **Rank position** is real: it's your app's actual index in a live search for that term.
- **Volume/difficulty** are heuristics (0-100) derived from result-count and competitor-authority
  proxies, not real search volume — there's no free source for that.
- iOS **subtitle** is scraped from the public App Store product page (not exposed by the iTunes
  API). If Apple changes their page markup this can silently start returning `null` — see
  `fetchSubtitle` in `src/lib/stores/appstore.ts`.
- Every store call (both platforms) retries transient failures (429s, 5xx, network errors) with
  backoff via `src/lib/withRetry.ts`, so one rate-limited request doesn't fail an entire daily
  tracking pass.

## Setup

```bash
npm install
docker compose up -d          # local Postgres on port 5433
npx prisma migrate dev        # create tables
npm run dev
```

Open http://localhost:3000, click **Add App**, search for your app, and add it.

Copy `.env.example` to `.env` and adjust `DATABASE_URL` if you're pointing at a different Postgres
instance (e.g. Supabase/Neon/RDS instead of the bundled docker-compose one).

## Daily tracking

Rank checks don't run automatically — trigger `POST /api/track` (protected by `CRON_SECRET` from
your `.env`) once a day:

```bash
curl -X POST http://localhost:3000/api/track -H "Authorization: Bearer $CRON_SECRET"
```

`scripts/daily-track.sh` wraps this for cron:

```cron
0 6 * * * APP_URL=http://localhost:3000 CRON_SECRET=... /path/to/scripts/daily-track.sh >> /var/log/aso-track.log 2>&1
```

Any external scheduler works the same way (GitHub Actions `schedule:` trigger, a hosting
provider's cron jobs, etc.) — it just needs to hit that one endpoint once a day.

### Or run the bundled cron container

`docker compose up -d cron` starts a small Alpine container (`docker/cron/`) that runs
`scripts/daily-track.sh` on a schedule via `crond`, hitting the app over `host.docker.internal`
since the app itself runs on the host, not in Docker. Configure it via `.env`:

```bash
CRON_SECRET="..."                        # required, shared with the app's own .env
APP_URL="http://host.docker.internal:3000"  # optional, this is the default
CRON_SCHEDULE="0 6 * * *"                # optional, standard 5-field cron syntax
```

## Connecting an agent (MCP)

The app exposes an MCP server at `/api/mcp` (Streamable HTTP transport) with tools for listing
apps, searching stores, adding apps/keywords/competitors, pulling health reports, finding winning
keywords, generating AI copy suggestions, and running a tracking pass on demand.

Point any MCP-compatible client at it, e.g. in Claude Code (`.mcp.json`, already checked in at the
repo root pointing at localhost):

```json
{
  "mcpServers": {
    "aso-copilot": {
      "url": "http://localhost:3000/api/mcp"
    }
  }
}
```

There's no auth on this endpoint by default — it has full read/write access to your ASO data. Set
`MCP_AUTH_TOKEN` in `.env` before exposing it beyond localhost, then have your client send it:

```json
{
  "mcpServers": {
    "aso-copilot": {
      "url": "http://localhost:3000/api/mcp",
      "headers": { "Authorization": "Bearer <MCP_AUTH_TOKEN>" }
    }
  }
}
```

## AI Copy Suggestions

Two independent ways to generate rewritten title/subtitle/description copy, in
`src/components/AICopySuggestions.tsx` on each app's Overview tab - pick either, both write to the
same `AiCopySuggestion` table so the panel shows whichever ran most recently:

- **OpenRouter** (works from the web UI): set `OPENROUTER_API_KEY` in `.env` (get one at
  [openrouter.ai/keys](https://openrouter.ai/keys)); optionally `OPENROUTER_MODEL` to pick a
  different model than the default. Click **Get AI suggestions**.
- **MCP, no key needed**: have an MCP-connected model call `prepare_copy_localization_brief` (returns
  current copy, character limits, and a localization instruction per locale - no network call, no
  key) and write the copy itself, then `save_copy_suggestions` to persist it. Useful if you're
  already paying for an agent and don't want a second LLM bill for this.

## Analytics (optional)

Two separate PostHog integrations - don't mix them up:

<img width="1558" height="521" alt="image" src="https://github.com/user-attachments/assets/04985484-c04c-4e2d-95c4-65c56c2f9e1c" />


- **This tool's own usage analytics** (are you clicking around this app, not any tracked app's
  users) — off by default. Set `NEXT_PUBLIC_POSTHOG_KEY` (and optionally
  `NEXT_PUBLIC_POSTHOG_HOST` if self-hosting PostHog) in `.env` to enable pageview tracking via
  `src/components/PostHogProvider.tsx` — no code changes needed, it's a no-op until the key is set.
- **Per-app product health** — each tracked app can link its *own* PostHog project (Settings tab,
  `src/components/PostHogSettings.tsx`) so its real daily active users show up next to its ASO
  health score on the Overview tab. Unrelated to the key above; configured per-app in the UI, not
  via env vars.

## Project layout

- `src/lib/stores/` — App Store (iTunes) and Google Play data fetching
- `src/lib/health.ts` — health score engine
- `src/lib/research.ts` — keyword candidate generation + volume/difficulty scoring
- `src/lib/ai.ts` / `src/lib/aiLocales.ts` — AI Copy Suggestions: OpenRouter prompt + call, the
  key-free localization-brief builder, and the 8-locale list
- `src/lib/reviewAnalysis.ts` — rating distribution + positive/negative theme extraction
- `src/lib/posthogIntegration.ts` — per-app PostHog client (daily active users query)
- `src/lib/storeLinks.ts` / `src/lib/metricColor.ts` — small shared UI helpers (outbound store
  links, volume/difficulty coloring) kept out of `research.ts` so Client Components importing them
  don't drag in server-only store-scraping dependencies
- `src/lib/appService.ts` — shared logic used by both the REST API and the MCP tools
- `src/app/api/` — REST routes
- `src/app/api/[transport]/route.ts` — MCP server
- `docker/cron/` — optional bundled cron container for daily tracking (see "Daily tracking" above)

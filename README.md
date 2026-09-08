# Newsflow — Watchlist edition (MGA)

A colourful, visual-first news dashboard that surfaces only **fundamental
business news** about a **tracked watchlist of companies** — orders, capex,
mergers, approvals, fraud, and so on — with day-to-day share-price noise
stripped out. Every story is source-backed with a real link.

This is the **watchlist-only** edition: there is a **single Watchlist feed**
(no Portfolio or Universe tabs, and no external company sync). The tracked
companies live in `public/data/companies.json`, and anyone can grow the list
from the **+ Add** box in the app. Emails are delivered under the **Munshot**
brand. There is **no login**.

> Everything degrades gracefully: no Bedrock key → clean neutral interim feed;
> no Munshot email secrets → the digest just doesn't send; no KV namespace →
> the add-boxes fall back to per-browser storage. See `email-preview.html` for
> the exact email look.

---

## The tracked watchlist (starting set)

| Company | NSE symbol | Sector |
| --- | --- | --- |
| Bliss GVS Pharma Limited | `BLISSGVS` | Healthcare |
| Jayaswal Neco Industries Limited | `JAYNECOIND` | Metals & Mining |
| Poojaa Precision Engg. Limited | `PPEL` | Automobile & Auto Components |
| Engineers India Limited | `ENGINERSIN` | Capital Goods |
| Larsen & Toubro Limited | `LT` | Capital Goods |

Edit this list any time in `public/data/companies.json` (under
`watchlist_exited`), or add companies live from the **+ Add** box in the app.
The NSE symbol is what the Filings tab uses to fetch that company's
announcements, so keep it accurate.

## Quick start

```bash
npm install
npm run dev
```

Open the printed URL — the dashboard loads immediately (no password).

| Script            | What it does                                        |
| ----------------- | --------------------------------------------------- |
| `npm run dev`     | Start the Vite dev server                           |
| `npm run build`   | Type-check + build the static site into `dist/`     |
| `npm run preview` | Preview the production build locally                |
| `npm run deploy`  | Build and deploy to Cloudflare Workers via wrangler |

## Tech stack

- **React 18 + TypeScript + Vite**, **Tailwind CSS 3**, **Recharts**, **lucide-react**
- **Cloudflare Workers** (via `wrangler`) serve the built site and the `/api/*` routes
- **Node ESM scrapers** (`/scrapers`) using Google News RSS + `fast-xml-parser`

## What you can do in the dashboard

- **Watchlist feed** — your tracked companies plus anything you add.
- **Pulse** — newsflow over the recent window, topic donut, "most in the news", mood.
- **Feed** — the news as filterable/sortable cards (topic, company, source, mood, importance, search).
- **Filings** — NSE announcements for your companies, filterable by company and category.
- **+ Add** — add your own companies and keywords.

## The data pipeline (`/scrapers`)

GitHub Actions runs the scrapers on a schedule and commits fresh JSON back to
`main`; the dashboard then reads `public/data/news.json` and
`public/data/filings.json`. Each push to `main` auto-deploys via Cloudflare.

```bash
cd scrapers
npm install
node news.mjs                 # gather candidates -> ../public/data/news.json
node enrich.mjs               # Claude keeps/drops + tags (no-op without a key)
node filings.mjs              # NSE announcements -> ../public/data/filings.json
ENRICH_MOCK=1 node enrich.mjs # test the enrich pipeline locally, no Bedrock key
```

**News (`news.mjs`)** — for each tracked company (watchlist + any custom stocks
from KV) one lightweight query per source: **Google News RSS** (free, no key),
the **Valuepickr** forum (no key; opinion-heavy, so Claude keeps only genuine
fundamentals), plus an optional **Munshot** supplement (`MUNS_TOKEN`) and
optional **Firecrawl** pass (`FIRECRAWL_API_KEY`). Recall over precision: it
keeps every item that mentions the company and isn't caught by the noise / SEO
blocklist. _Fallback with no Bedrock key:_ it requires a literal keyword so the
interim feed stays clean.

**Enrich (`enrich.mjs`) — the brain.** Sends each new item to **Claude on Amazon
Bedrock** and decides KEEP vs DROP, then sets `topic`, `mood`, `importance`, and
a plain `takeaway`. Junk (price moves, broker ratings, "stocks to watch", data
pages, SEO) is dropped. Only unenriched items are processed; missing a key is a
safe no-op.

**Filings (`filings.mjs`)** — fetches **real NSE corporate announcements**
directly (no key) using each company's NSE symbol. If NSE blocks the runner it
leaves the committed file untouched (never blanks it).

**Safety rules:** every item has a real source URL; results are de-duped and
merged into the existing file; a run that produces zero items never blanks the
file; existing enriched items are never re-clobbered.

## Email digest

Visitors subscribe from the **Brief** button (email + day/time). The Worker
stores each subscription in KV and an **hourly GitHub Actions job**
(`.github/workflows/send-digests.yml`, which POSTs to `/api/run-digests`) emails
everyone who's due — a **Munshot-branded newspaper** of their watchlist news,
sent once per day, with a one-click unsubscribe link. Without the Munshot email
secrets it runs and simply doesn't send.

---

## One-time setup (then automated forever)

Deployment is **not** handled here — connect the repo to **Cloudflare Workers
Builds** once and every push to `main` auto-deploys (Cloudflare runs
`npx wrangler deploy`, and the `[build]` hook builds the site first). The Worker
`name` in `wrangler.toml` is `mganews`.

**1. Cloudflare KV** (needed so the **+ Add** box persists and gets scraped, and
so email subscriptions are stored):

```bash
npx wrangler kv namespace create NEWSFLOW_KV
```

Paste the printed id into the `[[kv_namespaces]]` block in `wrangler.toml` and
uncomment those three lines, then push. Until then the add-boxes fall back to
per-browser storage and subscriptions won't persist.

**2. GitHub repo secrets** (repo → Settings → Secrets → Actions) — all optional;
add the ones you want:

| Secret              | Enables                                                       |
| ------------------- | ------------------------------------------------------------ |
| `BEDROCK_API_KEY`   | The Claude enrichment brain (Bedrock bearer token)           |
| `BEDROCK_MODEL_ID`  | The Claude model / inference-profile id (Haiku 4.5 rec.)     |
| `AWS_REGION`        | Bedrock region (defaults to `us-east-1`)                     |
| `NEWSFLOW_URL`      | Deployed site URL — lets the scraper read custom KV lists    |
| `MUNS_TOKEN`        | Munshot news supplement + the email digest send              |
| `MUNS_EMAIL`        | Munshot from-address                                         |
| `FIRECRAWL_API_KEY` | Firecrawl publisher pass (bonus)                             |
| `DIGEST_KEY`        | Shared secret for the hourly digest trigger (must match Worker) |

**3. Cloudflare Worker vars/secrets** (dashboard → the Worker → Settings, not
repo secrets): `MUNS_TOKEN` + `MUNS_EMAIL` + `MUNS_EMAIL_ENDPOINT` enable the
digest send; `DIGEST_KEY` unlocks `/api/run-digests`; `GH_DISPATCH_TOKEN`
unlocks the on-demand **Refresh** button; `SITE_URL` is a fallback origin for
unsubscribe links; `SEND_EMPTY=true` sends a "quiet day" note when there's no news.

To deploy manually: `npm run deploy` (after `wrangler login`).

## Automations

- **`refresh-news.yml`** — twice daily (08:01 / 20:01 IST) + on demand:
  news → enrich, commits `news.json` to `main` (auto-deploys).
- **`refresh-filings.yml`** — twice daily (08:15 / 20:15 IST) + on demand:
  NSE filings → commits `filings.json` to `main` (auto-deploys).
- **`send-digests.yml`** — hourly: triggers the email digest run.

To populate the dashboard immediately after setup, run the **Refresh News Data**
and **Refresh Filings** workflows once from the Actions tab (Run workflow).

## Project structure

```
public/data/
  companies.json     tracked watchlist companies (portfolio kept empty)
  keywords.json      the base keywords + 6-bucket mapping
  news.json          scraped news (envelope: generated_at, source, counts, items)
  filings.json       NSE filings (starts empty; scraper fills it)
scrapers/
  news.mjs           Google News + Valuepickr + Munshot + Firecrawl -> news.json
  enrich.mjs         Claude (Bedrock) keep/drop + topic/mood/importance/takeaway
  filings.mjs        NSE announcements -> filings.json
  email-preview.mjs  writes ../email-preview.html from the shared renderer
  lib/                fetch, RSS parse, keyword match, blocklist, merge, Valuepickr, Firecrawl
src/
  lib/               types, theme, data layer, metrics, storage, format, api
  components/         TopBar, Tabs, FilterBar, AddPanel, SubscribePanel, SourcesPanel, NewsCard, charts/, ui/
  pages/             Pulse.tsx, Feed.tsx, Filings.tsx
  App.tsx main.tsx index.css
worker/index.ts      Worker: serves dist/, /api/custom + /api/subscribe -> KV, digest sender
worker/email.mjs     newspaper email renderer (shared with the preview script)
.github/workflows/   refresh-news.yml, refresh-filings.yml, send-digests.yml
```

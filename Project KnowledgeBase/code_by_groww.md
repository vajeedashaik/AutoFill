# Groww Pulse (folder: `code-by-groww`) — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\code-by-groww`
**npm package name:** `smart-market-watchlist` v0.1.0 (the project was originally "Smart Market Watchlist" and was renamed **Groww Pulse** in commit `9aeb649`; the old name survives in `package.json` and the earlier `phaseN.md` / `context.md` text)
**Type:** Solo 72-hour hackathon web app (Next.js 15 full-stack), built in 10 documented phases. Local-demo-only; **not deployed** (deploy was explicitly skipped by the user in Phase 9).
**Git:** local repo on `master`, no feature branches, 94 commits. HEAD `1697719` ("perf: replace per-dot Framer Motion nodes with static SVG pattern fill"). Working tree at time of writing: `package-lock.json` modified, `phase10.md` untracked. README embeds a GitHub-hosted screenshot, but no remote is configured in the docs.

## 1. What this project is

A stock **watchlist that remembers exactly what you last saw** and only interrupts you when a change is *genuinely meaningful* — not just loud. Each change is scored against the stock's **own normal volatility** and the **market/sector move**, so a routine wobble on a choppy stock never crowds out a quiet-but-real move on a calm one. Optionally a user attaches a personal **thesis** ("why I own this") to a stock, and one AI call reads recent news to say whether the thesis still holds.

**Core insight (pitch line):** most market dashboards compete on more data / more charts. This one competes on **memory and attention** — it restores the state dashboards throw away between visits ("what did I already see?"), turning a wall of numbers into a short, honest "here's what changed."

Targets Indian (NSE, `.NS` symbols, INR display, Nifty 50 benchmark) and US-listed symbols. The Groww branding is the hackathon theme; the app is a standalone prototype, not affiliated code.

Main user flow: sign up → search & add stocks to watchlist (optional thesis/target) → background jobs snapshot prices every 5 min → user leaves → user returns to `/dashboard` "While you were away" digest with **Urgent / Notable / Routine** buckets → expand "Why is this flagged?" for the evidence trail → thesis verdict appears live for flagged stocks → set price/volume email alerts → drill into a stock's detail page with candlestick chart + company insights.

## 2. Tech stack

| Layer | Tech |
|---|---|
| Framework | Next.js 15.5 (App Router, Server Components, server actions, route handlers), React 19, TypeScript 5.7 |
| Styling / UI | Tailwind CSS v4 (config-less, `@import "tailwindcss"` + `@tailwindcss/postcss`), `motion` (Framer Motion) v13, `clsx` + `tailwind-merge`, custom "glassmorphism" dark theme, MagicUI-style components (`animated-shiny-text`, `blur-fade`, `border-beam`, `dot-pattern`, `magic-card`, `marquee`, `number-ticker`, `shimmer-button`) |
| Charts | `lightweight-charts` v5 (TradingView's open-source lib) for first-party candlesticks |
| Auth | **Clerk** (`@clerk/nextjs` 6) — native third-party auth for Supabase (JWT template is deprecated) |
| Database | **Supabase** Postgres + RLS (`@supabase/supabase-js` 2). Supabase project ref referenced in `simple.md`: `ngzxcsvemaqnafvbowsk` |
| Background jobs | **Inngest** v4 (`inngest`, `inngest-cli` for local dev dashboard on :8288) |
| Market data | `yahoo-finance2` v4 (primary; free, no key, covers NSE + US) and **Finnhub** REST (secondary, US-only on free tier; also news, search, company insights) |
| AI | **Gemini** via Inngest `step.ai.infer` (`gemini-flash-latest`), native structured-JSON output. The **only** LLM touchpoint in the product |
| Email | **Resend** v6 (price/volume alert emails; degrades to "logged but not sent" without a key) |
| Dev tooling | `tsx` for the `verify:*` scripts. **No jest/vitest** — verification is `tsc` + `next build` + pure-logic scripts + manual browser testing |

### Scripts (`package.json`)
`dev`, `build`, `start`, `lint`, `typecheck` (`tsc --noEmit`), `inngest` (`inngest-cli dev`), `verify:scoring`, `verify:digest`, `verify:thesis`, `verify:reconcile`. (`verify:alerts` and a GitHub Actions CI workflow were requested in `phase10.md` but **do not exist yet** — see §13.)

### Environment variables (`.env.example`; real values in gitignored `.env.local`)
- Clerk: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in`, `..._SIGN_UP_URL=/sign-up`, `..._SIGN_IN_FALLBACK_REDIRECT_URL=/dashboard`, `..._SIGN_UP_FALLBACK_REDIRECT_URL=/dashboard`
- Supabase: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` (server-only, bypasses RLS)
- `FINNHUB_API_KEY` (server-only; search falls back to a static NSE list without it)
- `GEMINI_API_KEY` (thesis relevance)
- `RESEND_API_KEY`, optional `ALERT_FROM_EMAIL` (defaults to Resend's `onboarding@resend.dev`), `NEXT_PUBLIC_APP_URL` (link in alert emails)
- Inngest Cloud only: `INNGEST_EVENT_KEY`, `INNGEST_SIGNING_KEY` (not needed locally)

## 3. Repository layout

```
code-by-groww/
├── app/
│   ├── layout.tsx, page.tsx (landing), globals.css, error.tsx, global-error.tsx, icon.png
│   ├── sign-in/[[...sign-in]]/, sign-up/[[...sign-up]]/      Clerk components
│   ├── (protected)/                                          layout does auth.protect()
│   │   ├── dashboard/  page.tsx, loading.tsx                 the digest
│   │   ├── watchlist/  page.tsx, loading.tsx, actions.ts, alert-actions.ts
│   │   └── stocks/[symbol]/page.tsx                          detail page (chart + insights)
│   └── api/
│       ├── search/route.ts                 stock search (Finnhub + NSE fallback)
│       ├── watchlist/diffs/route.ts        diff + score + thesis trigger, one batched call
│       ├── stocks/[symbol]/candles/route.ts  OHLC bars for the chart
│       ├── inngest/route.ts                Inngest serve endpoint (not Clerk-gated)
│       └── dev/trigger/route.ts            dev-only manual job trigger (404 in prod)
├── components/
│   ├── digest/        digest-view, bucket-section, stock-card, routine-line, newly-added-section, why-flagged-detail
│   ├── watchlist/     add-stock, remove-stock-button, edit-thesis, manage-alerts, price-cell, staleness-badge, diff-panel, time-machine, stock-chart-toggle
│   ├── stocks/        company-header, key-metrics-grid, analyst-ratings-chart, sentiment-card, interval-chart, insights-unavailable
│   ├── dashboard/     stat-row, sector-snapshot
│   ├── charts/        candlestick-chart.tsx
│   ├── magicui/       animated/visual effect primitives
│   ├── ui/            backdrop, badge, button, glass-card, logo, skeleton
│   └── header.tsx
├── lib/
│   ├── market-data/   index.ts (facade), types, staleness, reconcile, sectors, sources/{yahoo,finnhub}.ts
│   ├── scoring/       score, volatility, benchmarks, history, compute-for-diffs
│   ├── digest/        interpret, summarize, time-machine, insights
│   ├── thesis/        types, parse-verdict, prompt, trigger
│   ├── watchlist/     diff, snapshots, scored-diff, format-elapsed
│   ├── inngest/       client.ts, functions/{snapshot-ingest, daily-history-backfill, thesis-relevance, alert-check, shared}.ts
│   ├── alerts/types.ts, email/{resend, send-alert-email, alert-template}.ts
│   ├── stocks/        nse-fallback, types, insights (Finnhub company insights)
│   ├── news/finnhub-news.ts
│   ├── supabase/      server.ts, client.ts (hook), admin.ts (service role)
│   └── hooks/use-debounce.ts, utils.ts
├── scripts/           verify-scoring.ts, verify-digest.ts, verify-thesis.ts, verify-reconcile.ts
├── supabase/          schema.sql (combined dump) + migrations/0001..0006
├── types/database.ts  hand-written Supabase `Database` type
├── docs/superpowers/  specs/ (phase 1,3,5,6,7 designs) and plans/ (phase 3–7)
├── middleware.ts      clerkMiddleware route gate
└── *.md               README, context (living build log), phase1–10 briefs, DEMO_SCRIPT, base_guide, simple
```

## 4. Routes

| Path | Access | Purpose |
|---|---|---|
| `/` | public | Landing page: states the product + core insight, CTA |
| `/sign-in`, `/sign-up` | public | Clerk components |
| `/dashboard` | auth | The digest — "while you were away", Urgent/Notable/Routine, stat row, sector snapshot |
| `/watchlist` | auth | Search + add/remove, edit thesis, raw table with price/staleness/diff, Market Time Machine, per-row chart toggle, manage alerts |
| `/stocks/[symbol]` | auth | Detail page: interval candlestick chart, company header, key metrics, analyst ratings, news sentiment (Finnhub insights are US-only; NSE shows an "unavailable" card) |
| `GET /api/search?q=` | auth | Finnhub `/search` merged with 40-symbol NSE fallback, de-duped |
| `GET /api/watchlist/diffs` | auth | Computes diffs + scores for the whole watchlist in one call, fires capped thesis checks |
| `GET /api/stocks/[symbol]/candles?days=` | auth | OHLC bars (default 90, max 365) via Yahoo |
| `/api/inngest` | internal | Inngest sync/invoke |
| `/api/dev/trigger?job=snapshot\|history` | auth, dev-only | Fire jobs manually; 404 in production; GET is intentionally state-changing |

**Gating:** `middleware.ts` protects `/dashboard(.*)` and `/watchlist(.*)`; `app/(protected)/layout.tsx` re-checks with `auth.protect()` (defense in depth). **Known gap:** the middleware matcher still does **not** list `/stocks(.*)` (phase10 task 3) — not a security hole because the layout gate covers it, but the "defense in depth" claim is only true for two of three routes.

## 5. Database (Supabase Postgres, migrations `0001`–`0006`)

Identity is the **Clerk user id** (text) — `user_id text default (auth.jwt() ->> 'sub')`, so inserts need no explicit `user_id`. Clerk's session token is attached to Supabase clients via the `accessToken` option (native third-party auth; must be enabled in Supabase → Auth → Third Party Auth → Clerk).

| Table | Purpose | RLS |
|---|---|---|
| `watchlist_items` | `id, user_id, symbol, company_name (0003), thesis, target_price, added_at`, `unique(user_id, symbol)` | user-scoped select/insert/update/delete |
| `market_snapshots` | shared price history: `id, symbol, price, volume, source, fetched_at, status`; **Phase 8 adds** `conflict, alt_source, alt_price, alt_fetched_at` (0005) | select-only for authenticated; service role writes |
| `user_seen_state` | `(user_id, symbol)` PK, `last_seen_snapshot_id → market_snapshots`, `seen_at` — the "what I last saw" memory | user-scoped |
| `daily_history` | `(symbol, date)` PK, `close, volume` — feeds volatility + benchmarks | select-only for authenticated |
| `change_events` | scored detections: `user_id, symbol, detected_at, snapshot_id (0004), meaningfulness_score, magnitude, confidence, explanation jsonb, thesis_verdict`, `unique(user_id, symbol, snapshot_id)` | user-scoped |
| `alerts` | (0006) `user_id, symbol, company_name, alert_type ∈ {price_above, price_below, volume_above}, threshold, active, last_triggered_at, cooldown_minutes (default 60), created_at`; partial index on `symbol where active` | user-scoped |

Migrations: `0001_init` (tables), `0002_rls`, `0003_watchlist_company_name`, `0004_change_events_snapshot_dedup`, `0005_market_snapshot_conflict`, `0006_alerts`. `schema.sql` is the concatenation for a fresh project. `types/database.ts` is hand-written (header says it would be clobbered by `supabase gen types`).

**Client split:** `lib/supabase/server.ts` (RSC/actions, user-scoped via Clerk token), `client.ts` (`useBrowserSupabaseClient` hook), `admin.ts` (service role, server-only — used by Inngest jobs that write shared tables). State persistence claim: everything lives in Supabase keyed by Clerk user id, RLS-enforced; **zero `localStorage`/`sessionStorage`** in client code.

## 6. Market data pipeline (Phase 3)

- `lib/market-data/index.ts` is the **only** module consumers import (isolation boundary — nothing outside `sources/` touches `yahoo-finance2`/`finnhub.io`). Functions: `getQuote`, `getAllQuotes` (returns `{quotes, errors}` so the job can tell total failure from secondary-source failure), `getDailyHistory` (first source that answers), plus `getCandles` (Phase 10 chart).
- **Yahoo** (primary): quote + daily history; drops the in-progress today bar; 8s timeout; maps 429/not-found to `MarketDataError` codes.
- **Finnhub** (secondary): US symbols only (`supports()` false for `.NS`/`.BO`); key sent via `X-Finnhub-Token` header (not query string); 6s timeout; declines history (paid).
- **Staleness** (`staleness.ts`, pure): `FRESH < 2 min`, `DELAYED 2–10 min` (inclusive), `STALE > 10 min`; unparseable timestamp → STALE. Because the cron runs every 5 min, steady state is usually DELAYED — honest, not a bug.
- **Inngest jobs** (`lib/inngest/functions/`, registered in `app/api/inngest/route.ts`; inngest@4 API: `createFunction({id,name,triggers:[...]}, handler)`):
  - `snapshot-ingest` — cron `*/5 * * * *` + event `market/snapshot.requested`. Distinct watchlisted symbols in chunks of 5 with 1s gap; one `market_snapshots` row per source that answered; one bad symbol never aborts the batch; decays status of last-15-min rows; runs reconciliation (below).
  - `daily-history-backfill` — cron `30 1 * * *` + event `market/history.requested`; `HISTORY_DAYS = 60` calendar days (~44 trading days); upserts `daily_history` on `(symbol, date)`.
  - `loadWatchlistSymbols()` (in `shared.ts`) **unions** user symbols with the fixed reference set (Nifty `^NSEI` + 30 sector stocks), so benchmarks are fetched by the same jobs with no extra code.
- **Reconciliation** (`reconcile.ts`, pure, Phase 8): if two sources' fetch times differ by >60 s (`RECONCILE_TOLERANCE_MS`) prefer the newer (this tier is effectively unreachable in production since both are fetched together); else if prices differ by >0.1 % (`CONFLICT_THRESHOLD_PCT`) it's a **conflict**: both kept (every quote is its own row), tie-break = source priority **Yahoo, then Finnhub**, never averaged; the chosen row records `conflict/alt_source/alt_price/alt_fetched_at`. Only US symbols can conflict.

## 7. Seen-state & diffing (Phase 4)

- `user_seen_state` records, per user per symbol, the last snapshot shown.
- `lib/watchlist/diff.ts` → `computeDiffsForUser()`: exactly **3 queries** regardless of watchlist size (seen rows, latest snapshots, "then" snapshots by id) — no N+1. `isFirstView: true` when there's no seen row (instead of a fake zero delta). `SymbolDiff` also carries `currentSnapshotId`, `currentSnapshotFetchedAt`, `conflict/altSource/altPrice/usedSource`.
- `markWatchlistSeen()` (server action) takes **no snapshot id** — it re-queries the current latest snapshot per symbol server-side at write time (race-condition policy: never trust a client id). Upsert on `(user_id, symbol)` is idempotent.
- `WatchlistDiffsProvider` (client, `components/watchlist/diff-panel.tsx`) fetches `/api/watchlist/diffs` once, renders, and only *then* calls `markWatchlistSeen()`. `useWatchlistDiffs()` exposes the Map to digest components without a second fetch.
- Accepted gaps: a snapshot landing in the few-hundred-ms window between fetch and mark-seen is marked seen without ever being displayed; two near-simultaneous mark-seen calls (two tabs) can regress the pointer. Orphaned `user_seen_state` rows for removed stocks are ignored, not cleaned.
- `formatElapsed()` uses relative phrasing ("4 hours ago").

## 8. Meaningfulness engine (Phase 5) — the core algorithm

`lib/scoring/score.ts` → `computeMeaningfulness()`; all four signals are z-scores **divided by the stock's own 20-day daily volatility** (`dailyVolPct`), so "how unusual is this *for this stock*" drives ranking:

- `SCORE_WEIGHTS = { priceAnomaly: 0.4, volumeAnomaly: 0.2, marketRelative: 0.2, sectorRelative: 0.2 }`
- `BUCKET_THRESHOLDS = { urgent: 2.0, notable: 0.8 }` → bucket **Urgent ≥ 2.0**, **Notable ≥ 0.8**, else **Routine** (`deriveBucket()` exported for reuse on persisted scores)
- **Confidence** Low/Medium/High from data completeness (volatility available? sector mapped? volume available?). `data_completeness` has 8 values derived from the same three factors so it can't disagree with confidence. `isStale: true` forces confidence to **Low** (score untouched — staleness reduces trust, not the number).
- Throws on non-finite input (a silent NaN would misclassify as Routine — worst failure mode).
- Supporting math: `volatility.ts` (`computeVolatilityPct` — 20-day std dev of daily returns, null under threshold; `computeAverageVolume`), `benchmarks.ts` (latest-vs-prior-close daily move; sector benchmark = mean of sector members), `history.ts` (`loadRecentHistory`: one batched bounded query, 3× safety margin, deterministic secondary sort; known limitation: a shared global LIMIT could starve a symbol under uneven date coverage).
- `sectors.ts`: static map, **6 sectors × 5 NSE large-caps** (IT, Banking, Auto, Pharma, FMCG, Energy) + Nifty 50 (`^NSEI`) as market benchmark. Unmapped symbols get `sector_used: null` and degrade to market-only comparison (Medium confidence). Deliberate trade-off vs. a general sector taxonomy.
- Worked scenarios (verified in `verify-scoring.ts`): choppy 5 %/day stock in a +5 % rally moving +7 % → **0.68 / Routine**; calm 1 %/day stock, flat market, 4× volume, +3 % → **2.94 / Urgent**.
- `compute-for-diffs.ts` → `computeAndPersistScores()`: batches history once, memoizes sector benchmarks, per-diff try/catch (one bad symbol never aborts the batch), upserts `change_events` on `(user_id, symbol, snapshot_id)`.
- **Critical Phase 7 fix (immutability rule):** a `(symbol, snapshot_id)` that already has a `change_events` row is **read back and reused, never recomputed**. Without this, the 5 s thesis poll saw `then === current` (0 % delta), recomputed a near-zero score, and overwrote the row — silently demoting an Urgent card to Routine mid-session. Staleness/conflict are likewise frozen at first-scoring time. The poll merges **only the `thesis` field** onto already-rendered diffs.
- `explanation` jsonb shape: `price_change_pct, price_zscore, volume_ratio, market_change_pct, sector_change_pct, sector_used, data_completeness, stale, conflict, alt_source, alt_price` (+ `thesis_analysis` after Phase 7).

## 9. Digest UI (Phase 6)

- `/dashboard` (Server Component, `force-dynamic`) fetches `watchlist_items`, wraps `DigestView` in `WatchlistDiffsProvider`. `loading.tsx` shows a muted line during the server fetch.
- `lib/digest/summarize.ts`: `bucketDiffs()` → urgent/notable/routine/newlyAdded (a non-first-view diff with a missing bucket logs `console.error` and falls back to Routine so a stock is never lost); `summaryLine()` produces one of three headlines: **normal** "N meaningful changes across M stocks.", **calm** "Nothing meaningful changed since you last checked.", **first-visit** "N stocks added — here's your first look."
- `lib/digest/interpret.ts`: `interpretExplanation()` — ordered template rules (high volume + independent move / high volume + tracked / independent + normal volume / tracked + normal volume / no comparison data); constants `HIGH_VOLUME_RATIO=2`, `INDEPENDENT_MOVE_THRESHOLD_PP=1`.
- Components: `StockCard` (Urgent/Notable full card with price, % since last seen, interpretation, expandable detail), `RoutineLine` (compact; price delta muted to gray in Phase 9 to avoid a "red/green anxiety ticker"), `NewlyAddedSection`, `BucketSection` (native `<details>`, Urgent/Notable open by default, Routine collapsed), `WhyFlaggedDetail` (evidence trail: price move → z-score → market → sector → volume → confidence; the *only* place confidence is shown; also shows the Data-conflict block and Thesis analysis).
- Post-Phase-9 redesign added `lib/digest/insights.ts` (`digestStats`, sector snapshot aggregation — pure, nulls stay null), `components/dashboard/stat-row.tsx`, `sector-snapshot.tsx`.

## 10. Thesis + AI relevance (Phase 7)

The **one** place AI is used, deliberately isolated from the deterministic score (never merged into `meaningfulness_score`/`bucket`).

- Trigger (`lib/thesis/trigger.ts`, `selectThesisChecksToRun`): Urgent/Notable only, non-empty thesis, not already assessed, score-priority order, **capped at 5** per digest load.
- `app/api/watchlist/diffs/route.ts` fires Inngest events `thesis/relevance.requested` with idempotency id `${userId}:${symbol}:${changeEventId}`; each `inngest.send()` is individually try/caught (Phase 8 fix — an unreachable Inngest bus used to 500 the whole route).
- `thesis-relevance.ts`: fetches Finnhub company news (last 5 days, ≤5 articles, `[]` if none), calls Gemini through `step.ai.infer` (`MODEL = "gemini-flash-latest"` — earlier pinned `gemini-2.5-flash` was deprecated and silently made every check fall back; fixed during browser testing), uses native `responseMimeType: application/json` + `responseSchema`, parses defensively (`parseVerdictResponse` never throws; malformed → `"unavailable"`), persists `thesis_verdict` + `explanation.thesis_analysis`. Any error degrades to a stored `"unavailable"` verdict without affecting price/score.
- Verdict categories: `supports | contradicts | unclear | no_new_information | unavailable` (UI labels: Mostly intact / Contradicted / Unclear / No new information / Unavailable / Checking…).
- Client `diff-panel.tsx` polls every 5 s (60 s cutoff) while any thesis is unassessed and merges only the `thesis` field.
- `edit-thesis.tsx` + `updateWatchlistThesis()` action touch only `watchlist_items`, never `change_events` — past verdicts stay historically accurate.
- **Pitch answer (why AI only here):** everything else is deterministic and reproducible from raw numbers; judging whether a headline "supports" a free-text human reason has no formula, so it's an opt-in, cost-capped, degradable layer.

## 11. Staleness, conflicts, Market Time Machine (Phase 8)

- `StalenessBadge` next to every price: FRESH/DELAYED render as neutral "updated Xm ago"; only STALE gets a distinct (calm) badge.
- Conflict visibility in `WhyFlaggedDetail` (both values, which was used, tie-break reason).
- **Market Time Machine** (`lib/digest/time-machine.ts` + `components/watchlist/time-machine.tsx`): per-row `<details>` on `/watchlist` — before/after table (price, volume, sector move, market move) + one-line summary from the already-fetched diffs context (no new fetch/endpoint); first-view stock shows "nothing to compare yet".
- Failure-handling audit walked every external call (Yahoo 8 s, Finnhub quote/search/news 6 s, Gemini via Inngest). Fixes: `inngest.send` try/catch; `/api/search` key moved from query string to `X-Finnhub-Token`. Documented non-fix: no manual timeout around `step.ai.infer` (Inngest owns step timeouts).

## 12. Polish (Phase 9) and post-Phase-9 features (Phase 10 scope, commit `ddb65f7` "Preview-1")

**Phase 9:** removed the `/debug` page (it inserted/deleted a live TEST row per load); added `app/error.tsx` + `app/global-error.tsx` (none existed) and `loading.tsx` for dashboard/watchlist; rewrote landing page and `<meta description>`; rewrote README judge-facing with a "Key decisions" section; wrote `DEMO_SCRIPT.md` (~4m15s core path: problem → user leaves → the return → explain one change → thesis → reliability → closing line, with hard-question answers).

**Shipped later in one large undocumented commit (`ddb65f7`)** — full visual redesign plus three genuinely new features, which is why `phase10.md` exists (to document/review them retroactively):

- **Price/volume email alerts** — `alerts` table (`0006_alerts.sql`); `createAlert`/`removeAlert`/`toggleAlert` server actions in `app/(protected)/watchlist/alert-actions.ts` (validates type ∈ price_above/price_below/volume_above, threshold > 0; cooldown hard-coded to `MIN_COOLDOWN_MINUTES * 4` = 60 min — the 15-min "minimum" is not enforced anywhere else); `lib/inngest/functions/alert-check.ts` (cron `*/5 * * * *` + event `alerts/check.requested`; loads active alerts, latest snapshot per symbol preferring newest then Yahoo, `isTriggered()` (>=/<= thresholds; volume requires non-null), `onCooldown()` (`elapsed < cooldown_minutes*60000`), `resolveEmail()` via Clerk primary address falling back to the first address, sends via `lib/email/send-alert-email.ts` → Resend, then **always** stamps `last_triggered_at` even if email failed — deliberate, so a flaky provider can't cause a burst later). UI: `components/watchlist/manage-alerts.tsx`. Email HTML: `lib/email/alert-template.ts`. Without `RESEND_API_KEY` alerts still trigger and log, email skipped.
- **Company insights page** `/stocks/[symbol]` — `lib/stocks/insights.ts` (React `cache()`-wrapped, `server-only`, Finnhub profile/metrics/analyst recommendations/news sentiment; each field independently optional; NSE `.NS/.BO` symbols short-circuited because Finnhub returns zeroed bodies rather than 404s). Components under `components/stocks/`.
- **Candlestick charts** — `components/charts/candlestick-chart.tsx` (lightweight-charts), `interval-chart.tsx`, `stock-chart-toggle.tsx`, `/api/stocks/[symbol]/candles`.
- **Visual redesign** — dark glassmorphism (`glass-card`, `backdrop`, MagicUI components). Latest commits replaced ~3–4 k per-dot Framer Motion nodes with a static SVG pattern fill (perf fix, commit `1697719`) and restyled controls/enlarged charts.

## 13. Current status and known gaps (as of 2026-09-26)

Phases 1–9 complete with design specs/plans and code review; Phase 10's documented work is **only partly done**. A browser pass was done via chrome-devtools MCP (sign-up → watchlist → dashboard → Time Machine → thesis AI check → prod build → signed-out redirects) per `simple.md`.

Outstanding per `phase10.md` (verified against the tree):
- **No `docs/superpowers/specs/2026-09-05-phase10-alerts-insights-design.md`**, and `context.md` has **no Phase 10 section** (it stops at Phase 9). `context.md`'s own text also still says "9 phases" and lists the removed `/debug` in Phase 1.
- **No `scripts/verify-alerts.ts` / `npm run verify:alerts`**; `isTriggered`/`onCooldown` aren't exported for testing.
- **No `.github/workflows/ci.yml`** — the `verify:*` scripts, `typecheck`, `build` only run manually.
- **`middleware.ts` matcher lacks `/stocks(.*)`.**
- README routes table omits `/stocks/[symbol]` and alerts; README says migrations `0001`→`0005` (there's now `0006`); `DEMO_SCRIPT.md` has no alerts/insights beat; README Known Limitations still says the **30–50 stock performance test was never run** (no real numbers recorded).
- Phase 10's suggested review points: cooldown minimum not enforced beyond the create action; `resolveEmail` fallback to first address when no primary is set; `getCompanyInsights` cache is server-only Finnhub data (not per-user) so cross-user staleness isn't a privacy issue.

Other known limitations (documented in README): no pruning/indexes on `market_snapshots`/`daily_history` beyond primary keys (reads bounded with `.limit()`); static sector map; dual-source conflict only for US symbols; the `loadRecentHistory` shared-LIMIT trade-off; no test runner.

Manual items only the user can do (from `simple.md` / `context.md`): confirm Clerk/Supabase/Finnhub keys; apply migrations `0001`–`0006` in Supabase SQL editor; wire Clerk ↔ Supabase third-party auth; seed a demo account; trigger `/api/dev/trigger?job=snapshot` 2–3 times **and** `?job=history` once (without history, scores sit at Low confidence and Time Machine shows "No comparison data"); add a thesis (AI check only fires for Urgent/Notable stocks — may need a volatile day or a pre-taken screenshot); fill `[...]` placeholders in `DEMO_SCRIPT.md`; rehearse timed twice; take fallback screenshots; cold-start test; confirm a real alert email actually arrives once.

## 14. How to run locally

1. Node 20+; create Clerk app, Supabase project, Finnhub key, Gemini key (Resend optional).
2. Run `supabase/migrations/0001…0006` in order (or `schema.sql` once); enable Clerk as Supabase Third-Party Auth provider.
3. `cp .env.example .env.local` and fill values.
4. `npm install`, then two terminals: `npm run dev` (app :3000) and `npm run inngest` (Inngest dev UI :8288).
5. Sign in, add stocks on `/watchlist`, trigger the snapshot job (`/api/dev/trigger?job=snapshot` or the Invoke button) and the history job once.
6. Verify: `npm run typecheck`, `npm run build`, `npm run verify:scoring|digest|thesis|reconcile` (check counts at last run: scoring 24, digest 22, thesis 21, reconcile 15). Verify scripts use relative imports because `tsx` doesn't apply tsconfig path aliases.

## 15. Key design decisions (pitch-ready)

- **Monolith over microservices:** one Next.js app + Supabase + Inngest; Inngest supplies durable scheduled jobs with retries without new infrastructure.
- **Memory over data volume:** compare against the exact snapshot the user last saw, not "yesterday's close."
- **Volatility-normalized scoring** so ranking reflects unusualness-for-this-stock, market/sector-relative rather than raw magnitude.
- **Honesty about data quality:** staleness bands, forced Low confidence on stale data, conflicts kept and shown rather than averaged/hidden.
- **AI quarantined** to thesis relevance, capped (5/load), idempotent, degradable to "unavailable".
- **RLS everywhere:** user-scoped tables + shared read-only reference tables written only by the service role; Clerk `sub` as the identity claim.
- **Final numbers:** 5-min polling; reconcile tolerance 60 s / conflict 0.1 %; weights 0.4/0.2/0.2/0.2; thresholds 2.0/0.8; 20-day volatility window; 60 calendar-day history backfill.
- **Deviations kept consistently:** work done directly on `master`; no test runner; phases 6–9 built directly against fully specified briefs rather than per-task subagent dispatch, with a final `code-reviewer` pass each.

## 16. Documentation map inside the repo

- `README.md` — judge-facing summary, key decisions, run instructions, routes, known limitations
- `context.md` (~1164 lines) — living per-phase build log with deviations, code-review findings, verification status (last updated for Phase 9)
- `phase1.md`–`phase10.md` — original per-phase briefs (phase10 = retroactive documentation/hardening brief, untracked)
- `docs/superpowers/specs|plans/` — design specs (phases 1, 3, 5, 6, 7) and plans (3–7)
- `DEMO_SCRIPT.md` — timed demo narrative + hard questions
- `simple.md` — plain manual pre-demo checklist (14 steps)
- `base_guide.md` — analysis of a reference project ("Signalist", Next.js + MongoDB + TradingView embeds) covering 7 features (live-price streaming, candlestick charts, search, watchlist/alerts, background jobs, etc.) with build recipes; used as a benchmark showing which features such projects commonly leave unbuilt (alerts are real here)

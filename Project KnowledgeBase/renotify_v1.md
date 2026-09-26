# ReNotify V1 — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\renotify_v1` (a copy also appears as `ReNotify_V1` — same folder on case-insensitive Windows)
**GitHub origin:** `https://github.com/vajeedashaik/ReNotify_V1.git`
**Type:** Web-app MVP (React SPA + Supabase backend), built as the first version of the ReNotify product. Deployed via Firebase Hosting. The README calls it "Production Ready", but the code has several MVP-level shortcuts (see §12).
**Sibling projects in `My Projects`:** `RENOTIFY_MOBILE_APP` (Expo/React Native mobile version), `ReNotify Consolidated` (minor-project review documents: abstract, architecture doc, product vision, backlog, release plan, review deck, APK).

## 1. What this project is

ReNotify (originally named **Docscan / DocScan AI**; the old name survives in the README, page titles, `sw.js` and package metadata) is a web app that stops users from missing deadlines on warranties, service intervals, bills and subscriptions. Two ways in:

1. **Manual scan** — upload a photo of an invoice/warranty card/receipt; Google Cloud Vision OCRs it, rule-based extraction finds dates, and reminders are auto-created.
2. **Auto Email Bill Import** (the "Cred-like" feature) — user connects Gmail via read-only OAuth; a backend job finds bill/invoice emails, downloads attachments, runs the same OCR + date pipeline, and creates reminders with no manual upload.

On top of that: a real-time dashboard, web-push reminders, and a Claude-powered chat assistant for warranty/AMC/servicing questions. Currency default is INR and OCR is tuned for Indian documents (GSTIN/PAN, `DD/MM/YYYY`, Hindi + regional language hints), but the app is region-agnostic in code.

Public landing copy: *"ReNotify helps users automatically extract bills and receipts from Gmail and set smart reminders so they never miss a due date."* Support email shown on the pricing page: `renotify34@gmail.com`.

## 2. Tech stack

| Layer | Tech |
|---|---|
| Frontend | React 18.3, TypeScript 5.8, Vite 5.4 (`@vitejs/plugin-react-swc`, dev server port **8080**), Tailwind 3.4 + `tailwindcss-animate`, shadcn/ui (Radix primitives), React Router 6, TanStack Query 5 (client created but hooks mostly use raw Supabase calls), react-hook-form + zod, Recharts, Sonner + shadcn toaster, `react-helmet`, `date-fns`, lucide-react |
| Auth | **Clerk** (`@clerk/clerk-react`, `@clerk/themes`). Clerk user ID (string like `user_…`) is the identity everywhere. Clerk `PricingTable` is used for subscriptions/billing UI. |
| Backend | **Supabase**: Postgres + Realtime + Edge Functions (Deno). Project ref in `supabase/config.toml`: `uafwoaneffdlmnglzngz`. |
| OCR | **Google Cloud Vision** REST (`images:annotate`, `TEXT_DETECTION` + `DOCUMENT_TEXT_DETECTION`) called only from edge functions |
| LLM | **Anthropic Claude Haiku 4.5** (`claude-haiku-4-5-20251001`) called from two edge functions (`chat-assistant`, `llm-date-extract`) |
| Email | Gmail API, `gmail.readonly` scope, OAuth 2.0 auth-code flow with offline access |
| Push | Web Push (VAPID) via service worker `public/sw.js` + `send-push-notification` edge function |
| Hosting | Firebase Hosting (`firebase.json` serves `dist/` with SPA rewrite; `.firebaserc` default project `renotify-84f76`) |
| Tooling | ESLint 9, PostCSS/Autoprefixer, `lovable-tagger` (project was scaffolded with Lovable), bun/npm/pnpm lockfiles all present |

Scripts: `npm run dev` / `build` / `build:dev` / `lint` / `preview`. No test framework is installed.

## 3. Repository layout

```
renotify_v1/
├── src/
│   ├── main.tsx                 ClerkProvider (needs VITE_CLERK_PUBLISHABLE_KEY) → App
│   ├── App.tsx                  QueryClient, Tooltip, toasters, BrowserRouter, AuthProvider, routes
│   ├── contexts/AuthContext.tsx Wraps Clerk → { user, userId, userEmail, isSignedIn, signOut }
│   ├── pages/                   Landing, DashboardPage, OCRPage, Settings (AutoImport Bills), PricingPage,
│   │                            DocsPage, Auth, Signup, Subscription, GmailCallback, PrivacyPolicy,
│   │                            TermsOfService, NotFound  (+ legacy Dashboard.tsx, Index.tsx — not routed)
│   ├── components/
│   │   ├── ocr/                 OCRModule, FileUploadZone, ProcessingStatus, ResultsViewer, OCRDashboard
│   │   ├── email-import/        EmailImportSettings (connect/disconnect/sync), ImportedBillsList
│   │   ├── dashboard/           DocumentList, StatsGrid, ImportedBillsDashboardCard
│   │   ├── reminders/           RemindersList
│   │   ├── notifications/       NotificationPreferences
│   │   ├── chat/AIAssistant.tsx Chat UI → chat-assistant edge function
│   │   ├── layout/              Navbar, Header, Footer
│   │   └── ui/                  shadcn/ui components
│   ├── hooks/                   useOCR, useReminders, useNotifications, useDocumentMetadata, useOCRStats,
│   │                            useUserStatistics, useEmailImport, useOCRBillIntegration, useBrowserSupport
│   ├── lib/                     entity-extraction, date-extraction-pipeline, image-preprocessing,
│   │                            token-encryption, browser-support, seed-browser-data, utils
│   ├── integrations/
│   │   ├── supabase/            client.ts (anon client), types.ts (generated, partly stale)
│   │   └── email/               gmailService, billOCRIntegration, emailBillProcessor
│   └── types/                   ocr.ts, dashboard.ts, email-import.ts, browser-support.ts
├── supabase/
│   ├── config.toml              verify_jwt=false for chat-assistant, auth-gmail-token, ocr-extract
│   ├── functions/               9 edge functions (see §6)
│   └── migrations/              8 SQL migrations (see §7)
├── public/                      sw.js, logos (dark/light), ico, robots.txt
├── clerk-react/                 Standalone stock Clerk+Vite starter (scratch/reference, not part of the app)
├── *.md / *.txt / *.js docs     README, TECHNICAL_ARCHITECTURE, IMPLEMENTATION_SUMMARY, QUICK_START,
│                                EDGE_FUNCTIONS_DEPLOYMENT, BILL_OCR_DEPLOYMENT_GUIDE, GOOGLE_VISION_SETUP,
│                                OAUTH_FIX, production-deployment, autobillimport, renotify_date_extraction_prompt,
│                                MIGRATION_STATUS.js
└── firebase.json / .firebaserc / vite.config.ts / tailwind.config.ts / components.json / tsconfig*.json
```

## 4. Frontend

**Routes (`App.tsx`):** `/` Landing · `/ocr` OCRPage · `/dashboard` DashboardPage · `/pricing` · `/docs` · `/settings` (titled "AutoImport Bills") · `/auth/gmail-callback` · `/privacy-policy` · `/terms-of-service` · `/subscription` (wrapped in `<Header/>`) · `/auth` (Clerk `SignIn`) · `/sign-up` · `*` NotFound. Navbar links: OCR Scanner, Dashboard, AutoImport Bills, Pricing, Privacy Policy, Terms of Service.

**Auth gating:** pages use Clerk's `<SignedIn>/<SignedOut>` and redirect signed-out users to `/`. Landing auto-redirects signed-in users to `/dashboard`. Theme (light/dark) toggles a class on `<html>`; Clerk components take `dark`/`light` base themes accordingly.

**DashboardPage tabs:** Overview (document list, quick actions, tip) · Reminders (`RemindersList`) · Imported Bills (`ImportedBillsDashboardCard`) · AI Assistant (`AIAssistant`, given a text summary of the user's top 10 documents as context) · Settings (account info, stats, an email-reminder checkbox that is not wired to anything, a "Delete Account" button that does nothing). Stats cards show total documents and upcoming reminders. It subscribes to Realtime changes on `ocr_jobs` (filtered by user) to refresh stats.

**Legacy/unrouted:** `pages/Dashboard.tsx` and `pages/Index.tsx` are old Docscan-era pages not referenced by the router. Notably, `Dashboard.tsx` is the only place that mounts `useNotifications()` (the periodic notification checker + push subscription), so in the currently-routed app that hook is effectively not running (see §12).

## 5. Core flows

### 5.1 Manual scan (`hooks/useOCR.ts`, `OCRModule`)
1. `FileUploadZone` collects files → `processSingleFile` per file.
2. **Preprocess** (images only, `lib/image-preprocessing.ts`): quality analysis (contrast/noise/sharpness), optional contrast/denoise/sharpen, resize to max 2048×2048.
3. **OCR:** base64 → `supabase.functions.invoke('vision-ocr')` → Google Vision → `{ fullText, confidence, textAnnotations }` (confidence is a fixed 0.95 if any annotations, else 0.7 — not Vision's real per-word confidence).
4. **Extraction** (`lib/entity-extraction.ts`, all client-side regex/keyword rules): `detectDocumentType` (keyword scoring across invoice/bill/warranty_card/receipt/product_manual/service_document), `extractDocumentTitle`, `extractVendorDetails`, `extractProductDetails`, `extractAmounts`, `extractDateDetails`.
   - `extractDateDetails` assigns per-line priorities: warranty/expiry 100, extended warranty 90, next service/service due 85, renewal/subscription 80, payment due 75, purchase 50, invoice date 45. Fallbacks: earliest/last future date becomes warranty expiry; purchase date estimated as warranty expiry − 1 year.
5. **Invoice-date pipeline** (`lib/date-extraction-pipeline.ts`, see §5.3) runs on top; if confidence ≥ 0.3 its date overrides `invoiceDate`, and a low-priority "Document Date" reminder is always added.
6. `generateReminderSuggestions`: warranty_expiry (priority by days left: ≤30 high, ≤90 medium), service_due (≤14 high, ≤30 medium), and for `bill` docs a `payment_due` = invoice date + 30 days (assumed payment term).
7. **Persist** (only if signed in): insert `ocr_jobs` → `ocr_results` (raw text + JSONB details) → `document_metadata` (title as `vendor_name`, expiry/renewal dates, category tag) → update `user_statistics` → `createRemindersFromOCR` (`notify_before_days` = 14/7/3 for high/medium/low). Dashboard refetches via `onScanComplete`.

### 5.2 Auto Email Bill Import
1. **Connect** (`EmailImportSettings`): browser redirects to Google OAuth with `scope=gmail.readonly`, `access_type=offline`, `prompt=consent`, redirect `/auth/gmail-callback`. (The `state` param is a random string that is never validated on return.)
2. **Callback** (`GmailCallback.tsx`) posts `{ code, userId (Clerk), redirectUri }` to edge function `auth-gmail-token`, which exchanges the code, reads the Gmail profile for the address, and upserts `email_imports` (one row per user, `enabled=true`, tokens stored base64-encoded, expiry timestamp).
3. **Sync** (`sync-email-bills`, cron `0 */6 * * *` per its `config.toml`, also manually triggered from the UI "Sync Now" via an unauthenticated `POST`): for every enabled import → refresh token if expiring (5-min buffer; on refresh failure it disables the import with an error message) → Gmail search over the **last 30 days**, max 20 messages, `subject:(invoice OR bill OR receipt OR warranty OR service OR renewal OR subscription OR expiry OR due OR payment) OR from:(invoice OR billing OR noreply OR support)` → skip already-imported `gmail_message_id` → pick first PDF/image/doc attachment (`gmail://<msgId>/<attachmentId>`) or, failing that, a bill-looking link in the top-level body → require both a file/link **and** bill keywords in subject/from → insert `imported_bills` → immediately run inline OCR + due-date extraction + reminder creation for that bill.
4. **Manual "Process"** button on `ImportedBillsList` → `useOCRBillIntegration` → edge function `process-bill-ocr` (re-downloads attachment with retry on "invalid attachment token", **images only — PDFs are rejected with an error**, runs Vision, stores `ocr_jobs`/`ocr_results`, links to `imported_bills`). Realtime subscription on `imported_bills` pushes new/updated bills to the UI with a toast.
5. **Disconnect** flips `enabled=false` (`disconnectEmailImport`). `autobillimport.md` specified deleting tokens on disable — check `gmailService.ts` if that matters.

Known behavior gaps in the inline sync-time OCR (`runOcrForImportedBill`): it sends attachment bytes straight to Vision's image endpoint (PDFs will likely fail there too), uses a simpler keyword-window date heuristic (not the 6-stage pipeline), hard-codes confidence 0.85, and creates a `payment_due` reminder with `notify_before_days=7` only. The originally specified 7/3/1-day multi-reminder scheme is not implemented.

### 5.3 Invoice-date extraction pipeline (`lib/date-extraction-pipeline.ts`)
Designed from the spec in `renotify_date_extraction_prompt.md`. Six independently exported stages behind `extractInvoiceDate(ocrText) → { invoice_date, confidence, reason }`:
1. `extractDateCandidates` — regex for DD/MM/YYYY (`/ - .`), YYYY-MM-DD, `DD MMM YYYY`, `MMM DD, YYYY`, plus an OCR-noise-corrected pass (O→0, l→1, S→5, I→1); timestamps stripped first; calendar-validated.
2. `scoreByContext` — ±10-word window: +3 "invoice date/bill date", +2 "date/purchase/issued/order", −3 "expiry/warranty end/valid till", −2 "due date/pay by".
3. `applyPositionalHeuristics` — +1 top 30% of doc, +1 repeated, +1 past date, +2 same line as a strong keyword.
4. `rankAndSelect` — highest score; ties → most recent past date.
5. `computeConfidence` — keyword strength (≤0.4) + gap to runner-up (≤0.3) + fewer-candidates bonus (≤0.3).
6. `llmFallback` — if confidence < 0.6 or no candidates, calls edge function `llm-date-extract` (Claude Haiku, `max_tokens 64`, first 3000 chars, strict JSON) and returns confidence 0.6 / reason `llm_fallback`; silently degrades if unavailable.
If nothing works it returns today's date with confidence 0.05.

### 5.4 Reminders & notifications
- `useReminders` — CRUD on `reminders` (filtered by Clerk `user.id`, non-dismissed, ordered by date), Realtime channel per user with toasts on INSERT, "upcoming" = within 30 days ahead / 7 days overdue, a load-time toast for reminders inside their notify window.
- Reminder types: `warranty_expiry`, `service_due`, `subscription_renewal`, `payment_due`, `custom`.
- `useNotifications` (when mounted): registers `/sw.js`, asks notification permission, subscribes to Web Push using `VITE_VAPID_PUBLIC_KEY`, stores the subscription in `push_subscriptions`; every 5 minutes checks reminders whose notify window has started (or ≤7 days overdue), shows a service-worker notification, invokes `send-push-notification` (works with the app closed), shows a toast, and sets `is_notified=true`. Preference toggles are upserted to `notification_preferences` but the hook returns hard-coded defaults rather than reading them back.
- `sw.js` shows push notifications (View/Dismiss actions, opens `/dashboard`).

### 5.5 AI assistant
`AIAssistant` → edge function `chat-assistant` → Claude Haiku 4.5 (`max_tokens 1024`) with a system prompt scoped to warranty/AMC/servicing/electronic-parts/bill-date questions. Document context (first 1500 chars) is prepended only to the first user message. Requires `ANTHROPIC_API_KEY` as a Supabase secret.

## 6. Supabase Edge Functions (Deno)

| Function | Called by | Purpose |
|---|---|---|
| `vision-ocr` | `useOCR` (manual scan) | Base64 image → Vision API → `{fullText, confidence, textAnnotations}`. Reads `GOOGLE_VISION_API_KEY` or `VITE_GOOGLE_VISION_API_KEY`. |
| `auth-gmail-token` | `GmailCallback` (raw `fetch`, no auth header) | OAuth code → tokens, Gmail profile, upsert `email_imports`. `verify_jwt=false`. |
| `sync-email-bills` | Cron (6h) + "Sync Now" button | Gmail scan for all enabled users, insert `imported_bills`, inline OCR + reminder. Uses service-role key. |
| `process-bill-ocr` | `ImportedBillsList` "Process" button | On-demand OCR for one imported bill (images only), token refresh, stores results. |
| `ocr-process-bill` | Nothing in current UI (older design) | OCR from a Supabase Storage `bill-documents` bucket path. Superseded by `process-bill-ocr`. |
| `ocr-extract` | Nothing in current UI (older design) | Earlier LLM/vision-prompt-based structured extraction (large system prompt returning JSON with vendor/product/dates/amount, priority on dates; 10 Indian languages). `verify_jwt=false`. |
| `llm-date-extract` | `date-extraction-pipeline` stage 6 | Claude Haiku invoice-date fallback. |
| `chat-assistant` | `AIAssistant` | Claude Haiku chat. `verify_jwt=false`. |
| `send-push-notification` | `useNotifications` | Reads `push_subscriptions` for a user, signs a VAPID JWT (ES256), POSTs to each endpoint, deletes subscriptions that return 404/410. Payload is sent as plain UTF-8 bytes with `Content-Encoding: aes128gcm` but is **not actually encrypted** per RFC 8291, so real push services will likely reject it. |

**Secrets/env used by functions:** `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `VITE_GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `APP_URL`, `TOKEN_ENCRYPTION_KEY` (read but effectively unused), `VITE_GOOGLE_VISION_API_KEY`/`GOOGLE_VISION_API_KEY`, `ANTHROPIC_API_KEY`, `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` (default `mailto:admin@renotify.app`).
**Frontend env (`VITE_*`):** `VITE_CLERK_PUBLISHABLE_KEY`, `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_GOOGLE_CLIENT_ID`, `VITE_VAPID_PUBLIC_KEY`. `.env.production` in the repo is empty (0 bytes); no secrets were found committed in what was read.

## 7. Database (Supabase Postgres)

Migrations, in order:
1. `20251204…` — `ocr_jobs`, `ocr_results` (JSONB: extracted_data, vendor/product/date details, reminder_suggestions, metadata; `confidence NUMERIC(5,4)`), `ocr_errors`, `ocr_statistics` + triggers; open public RLS.
2. `20251205…` — `profiles` (tied to `auth.users`), `handle_new_user` trigger, `ocr_jobs.user_id UUID`, per-user RLS (Supabase Auth era).
3. `20251208…` — first `reminders` table (UUID `user_id` → `auth.users`, CHECKed `reminder_type`, `auth.uid()` RLS).
4. `20251210_browser_support.sql` — `browser_support`, `web_features`, `feature_support`, `browser_stats` (a "caniuse replacement"; feature is unrelated to the product core — see §9).
5. `20251210_dashboard_tables.sql` — `reminders` (`CREATE IF NOT EXISTS`, `user_id TEXT`), `user_profiles`, `document_categories`, `document_tags`, `document_metadata`, `user_activity_log`, `document_exports`, `notification_preferences`, `user_statistics`, `subscription_plans`, `user_subscriptions`, `monthly_usage`, indexes and permissive RLS.
6. `20251211_fix_ocr_jobs_user_id.sql` — converts `ocr_jobs.user_id` to TEXT for Clerk IDs; RLS relaxed to `USING (true)`.
7. `20251216_email_imports.sql` — `email_imports` (one per user: provider `gmail`, address, `enabled`, `oauth_token`, `oauth_refresh_token`, `token_expires_at`, `last_synced_at`, `sync_error`) and `imported_bills` (`gmail_message_id`, subject, from, `file_url`, `file_type` attachment|link, `ocr_job_id`, `ocr_result_id`, `extracted_due_date`, `reminder_created`; `UNIQUE(user_id, gmail_message_id)` for de-dupe). Permissive RLS.
8. `20260331_push_subscriptions.sql` — `push_subscriptions` (endpoint unique, p256dh, auth) with RLS on `auth.uid()::text`.

**Core relationships:** `ocr_jobs 1—N ocr_results`; `ocr_results 1—1 document_metadata`; `ocr_results 1—N reminders`; `email_imports 1—N imported_bills`; `imported_bills → ocr_jobs/ocr_results`.

**Identity model:** Clerk user IDs are stored as TEXT `user_id` in every user-owned table. The Supabase client is a plain anon-key client with no Clerk-issued JWT, so `auth.uid()` is always null.

**Schema-drift caveat:** migration 3 creates `reminders` with a UUID `user_id`, and migration 5 uses `CREATE TABLE IF NOT EXISTS` — so if migration 3 ran first on a database, the TEXT-ID version never applied. The generated `integrations/supabase/types.ts` is also partly stale. Verify the live schema before trusting these files.

## 8. Security & privacy posture

Intended: read-only Gmail scope, per-user data isolation, user can disable import at any time, no email content stored unless a bill is detected, HTTPS.

Actual state (important if this is ever shown to real users):
- **RLS is effectively off for most tables.** Since Clerk IDs can't satisfy `auth.uid()`, policies were changed to `USING (true)` with "application will filter by user_id". Anyone with the public anon key can read or modify any user's rows, including `email_imports` (which holds OAuth tokens).
- **OAuth tokens are only base64-encoded**, not encrypted (`btoa`), in `auth-gmail-token`, `sync-email-bills`, `process-bill-ocr` and `lib/token-encryption.ts` (which itself says "NOT cryptographically secure"). `TOKEN_ENCRYPTION_KEY` is read but never used.
- **Edge functions trust caller-supplied `userId`** (e.g. `process-bill-ocr` takes it in the body; several have `verify_jwt=false`), and `sync-email-bills` can be triggered by an unauthenticated POST that processes *all* users.
- OAuth `state` is generated but never verified (CSRF gap); CORS is `*` on all functions.
- Vision API key is passed as a query string from server-side functions only (not in the browser bundle), which is correct; `production-deployment.md` documents a hardening checklist (Firebase App Check, key restrictions, Clerk/Supabase dashboard settings) that appears to be instructions rather than completed work.
- Privacy Policy and Terms of Service pages exist (needed for Google OAuth verification of the Gmail scope).

## 9. Miscellaneous / peripheral pieces

- **Browser support DB:** `lib/browser-support.ts`, `seed-browser-data.ts`, `useBrowserSupport.ts`, `types/browser-support.ts`, migration 4 and `MIGRATION_STATUS.js` implement a Supabase-backed browser/feature-support tracker (replacing caniuse). It is a leftover of the earlier "Docscan" scope and not part of the product flow.
- **Subscriptions:** billing UI is Clerk's `PricingTable`; `subscription_plans`/`user_subscriptions`/`monthly_usage` tables exist (with a `stripe_subscription_id` column) but no code enforces plan limits.
- **`emailBillProcessor.ts`** (client-side email processing/scheduling) and parts of `gmailService.ts` (`fetchEmailsFromGmail`, `downloadGmailAttachment`) are older client-side designs; the working path is server-side in the edge functions.
- **`clerk-react/`** is an untouched Clerk+Vite starter folder.
- **Docs in repo:** README (feature/architecture overview, still titled Docscan, last updated 2025-12-23), `TECHNICAL_ARCHITECTURE.md` (pipeline/hook diagrams; describes the Docscan-era design), `IMPLEMENTATION_SUMMARY.txt` (email-import steps 5–9), `QUICK_START.md`, `EDGE_FUNCTIONS_DEPLOYMENT.md`, `BILL_OCR_DEPLOYMENT_GUIDE.md`, `GOOGLE_VISION_SETUP.md` (fixing a 401 from restricted API keys), `OAUTH_FIX.md`, `production-deployment.md`, `autobillimport.md` (the original feature prompt), `renotify_date_extraction_prompt.md` (the original spec for the date pipeline). These are mostly AI-assistant prompts/status notes; they describe intent and sometimes lag behind the code.

## 10. History (from git log, most recent first)

`Merge remote changes and resolve README conflict` → `added LLM ChatBot` (Claude assistant) → merge of PR #1 `feature/google-calendar-integration` (a branch/PR exists under this name; no Google Calendar code was found in `src`, so verify what it changed) → README cleanups → `token expiry issue fixed` → `basic debugging` → `final mvp v1` → `process ocr bill from Gmail` → `auto-bill-import implemented` → `Auto Email Import Added` → `final mvp created` → `google cloud vision implemented` → earlier Clerk/RLS/CORS fixes and dashboard real-time work (Dec 2025). Later additions visible in files: date-extraction pipeline + `llm-date-extract`, push notifications (migration dated 2026-03-31), Claude chat assistant. Work is by the repo owner `vajeedashaik`. (A full `git log` count/date range was not retrieved because the command timed out.)

## 11. Running it locally

```bash
cd renotify_v1
npm install                      # (or pnpm / bun)
# create .env.local with:
# VITE_CLERK_PUBLISHABLE_KEY, VITE_SUPABASE_URL, VITE_SUPABASE_PUBLISHABLE_KEY,
# VITE_GOOGLE_CLIENT_ID, VITE_VAPID_PUBLIC_KEY
npm run dev                      # http://localhost:8080  (README says 5173 — vite.config.ts uses 8080)

# backend
supabase db push
supabase secrets set VITE_GOOGLE_CLIENT_ID=… GOOGLE_CLIENT_SECRET=… APP_URL=… \
  VITE_GOOGLE_VISION_API_KEY=… ANTHROPIC_API_KEY=… VAPID_PUBLIC_KEY=… VAPID_PRIVATE_KEY=…
supabase functions deploy vision-ocr auth-gmail-token sync-email-bills process-bill-ocr \
  llm-date-extract chat-assistant send-push-notification

# frontend deploy
npm run build && firebase deploy --only hosting
```
Google OAuth needs `http://localhost:<port>/auth/gmail-callback` (and the production URL) as authorized redirect URIs; in Testing mode, add users under OAuth consent screen → Test users. Note the port mismatch above when registering the redirect URI.

## 12. Known issues, risks and loose ends (things a future session should know)

1. **Security gaps in §8** are the biggest blocker to real production use (open RLS, base64 "encryption", unauthenticated sync, trust of client `userId`).
2. **Web Push likely doesn't work end-to-end:** payload isn't RFC 8291-encrypted, and `useNotifications` is only mounted in the unrouted legacy `Dashboard.tsx`, so neither push subscription nor the 5-minute notify loop runs in the live app. Only `useReminders`' toasts and Realtime updates are active.
3. **PDF bills aren't handled:** Gmail bills are very often PDFs, but both OCR paths send raw bytes to Vision's image endpoint (`process-bill-ocr` explicitly rejects PDFs). Vision's PDF path (`files:annotate` / async batch) isn't used.
4. **Email sync limits:** 30-day window, max 20 messages per run, first attachment only, top-level body links only (not multipart HTML), whole-inbox `from:(noreply OR support)` matches can pull noise; the spec's 60-day window and 7/3/1-day reminder ladder aren't implemented.
5. **Date heuristics are English/Indian-format-centric** and rule-based; the invoice-date pipeline has a bug-prone edge (`correctOCRNoise` blanket-replaces letters, and DD/MM vs MM/DD is always assumed DD/MM). `parseDate` helpers in `useOCR`/`useReminders` try `new Date()` first, which parses ambiguous formats as US-style.
6. **Duplicated/dead code:** duplicate `metadata` key in the `ocr_results` payload in `useOCR`; two parallel bill-OCR edge functions plus an unused storage-bucket design; unused `emailBillProcessor`; `useNotifications` returns static default preferences.
7. **Naming drift:** product is ReNotify but many titles/docs still say Docscan/DocScan AI; `package.json` name is `vite_react_shadcn_ts`.
8. **Schema drift risk** between migrations 3 and 5 and stale generated types (§7).
9. **Dashboard "Settings" tab** has non-functional controls (email reminders checkbox, Delete Account); no plan enforcement despite pricing UI.

## 13. Where to look for what

| Question | File(s) |
|---|---|
| How a scan is processed end to end | `src/hooks/useOCR.ts` |
| How dates/reminders are derived | `src/lib/entity-extraction.ts`, `src/lib/date-extraction-pipeline.ts` |
| Gmail connect flow | `src/components/email-import/EmailImportSettings.tsx`, `src/pages/GmailCallback.tsx`, `supabase/functions/auth-gmail-token/index.ts` |
| Background bill sync | `supabase/functions/sync-email-bills/index.ts` (+ its `config.toml` cron) |
| Manual bill OCR | `src/integrations/email/billOCRIntegration.ts`, `src/hooks/useOCRBillIntegration.ts`, `supabase/functions/process-bill-ocr/index.ts` |
| Reminder data + realtime | `src/hooks/useReminders.ts` |
| Push notifications | `public/sw.js`, `src/hooks/useNotifications.ts`, `supabase/functions/send-push-notification/index.ts` |
| Chat assistant | `src/components/chat/AIAssistant.tsx`, `supabase/functions/chat-assistant/index.ts` |
| DB schema | `supabase/migrations/*.sql` (verify against the live DB) |
| Deploy steps | `EDGE_FUNCTIONS_DEPLOYMENT.md`, `production-deployment.md`, `firebase.json` |

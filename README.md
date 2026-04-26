# Spendly (FinanceFlow)

> **Privacy-first personal finance for India — your bank SMS becomes a clean spend ledger without ever leaving your phone unencrypted.**

![stack](https://img.shields.io/badge/stack-React_Native_%2B_Node_%2B_Postgres_%2B_Ollama-1f6feb)
![mobile](https://img.shields.io/badge/mobile-React_Native_0.74-61dafb)
![backend](https://img.shields.io/badge/backend-Node_18%2B%20Express%20%2B%20Prisma_6-339933)
![db](https://img.shields.io/badge/db-PostgreSQL-336791)
![ai](https://img.shields.io/badge/ml-Mistral_7B_(local)-purple)
![license](https://img.shields.io/badge/license-TBD-lightgrey)

---

## 1. The Problem

Indian consumers get an SMS for every UPI, debit-card, and credit-card transaction, but the only ways to track that spend today are: type each line into a spreadsheet, hand bank credentials to an aggregator, or grant a third-party app cloud SMS sync. The first is abandoned in week two the latter two trade privacy for convenience and have repeatedly been breach targets.

The result is that most people fly blind on monthly spend until the credit-card bill arrives.

## 2. The Solution

Spendly reads the SMS that already exists in the device inbox, parses the transaction on the phone, and only ships the structured fields (amount, merchant, type, date) — never the raw, unredacted SMS body — to a backend the user already controls. Where the regex parser is uncertain, the transaction is flagged "unverified" instead of silently dropped, and labelled samples flow through a separate Python trainer that fine-tunes a local Mistral-7B model on Indian bank SMS specifically. The user gets an automatic spend ledger the data stays inside a private trust boundary.

## 3. Key Features

**Auto-capture & parsing**
- On-device SMS reader (Android `READ_SMS`) and real-time `BroadcastReceiver` — _new transactions appear seconds after the bank SMS lands_.
- Regex parser for 30+ Indian bank/UPI/wallet senders with confidence scoring — _high-recall on HDFC, SBI, ICICI, Axis, Kotak, Paytm, PhonePe, GPay, etc._
- Low-confidence parses surface as "unverified" instead of being dropped — _user reviews edge cases instead of losing them_.
- OTP digit masking + SHA-256 hashing before any upload — _live OTPs never reach the server, even if a row leaks_.

**Money management**
- Default + custom categories (10 seeded, scoped per user, EXPENSE/INCOME kinds). Defaults are deletable deletes are blocked server-side with a 409 + descriptive error if any transaction still references the category.
- Merchant → category mappings with optional per-merchant nickname — _"DMART AVENUE SUPERMARTS" displays as "Groceries Near Home"_. Saving a mapping retroactively categorizes every still-uncategorized transaction from the same merchant (kind-filtered, never overwrites a manual choice).
- Income flow has full category support — `Add Income` shows only INCOME-kind categories (Salary, Freelance, Side Projects), `Add Expense` shows only EXPENSE-kind.
- Manual transactions are date-bounded to today/past — no future-dated entries.
- Investments incl. SIP tracking with scheduled local notifications — _reminders without a server cron_.
- Manual transactions, edit, delete, and batch import. Deleting a transaction auto-pops back to the previous list rather than landing on a "not found" screen.

**Insights**
- Category breakdown, monthly totals, and rolling trend charts (Skia line + pie).
- Home `Insight` card is one tappable surface with four states (`warming-up` / `uncategorized` / `top-category` / `no-data`) below 20 categorized transactions it shows progress, otherwise it surfaces the top 1–2 uncategorized merchant names by frequency and deep-links to a pre-filtered Transactions list. Resolution mirrors `getTransactionDisplay` so the count always matches the visible rows (mappings count, not just direct `categoryId`).
- Daily / weekly summary notifications fire with rich data baked in at schedule time (`You spent ₹X today\nTop: Y ₹Z`, weekly carries the ↑/↓ delta vs last week). Tapping deep-links to a `TodayAnalysis` screen scoped to the snapshot date — a Sunday-night daily tapped on Monday morning still shows Sunday, not "today".
- Budget threshold alerts via local notifications.

**Sync & reliability**
- `SyncJob` rows track multi-batch imports with progress, errors, and resume — _the bar at the top of the home screen mirrors backend state, not optimistic UI_.
- Server-computed SHA-256 dedup key over `amount | YYYY-MM-DD | cardLast4 | cleanMerchant` — _eliminates Postgres "NULL is distinct" duplicate bugs across devices and timezones_.
- 90-day login-merge: hashes are primed from existing transactions on login so a fresh install never re-creates rows.

**Auth & onboarding**
- WhatsApp OTP (BotBiz delivery, code generated and verified server-side with HMAC-SHA256) plus Google Sign-In via Firebase.
- Two-token session: short-lived JWT access + opaque refresh in a DB `Session` row — _logout kills both tokens immediately_.

**ML training loop (separate trainer)**
- Backend buckets labelled SMS into `TrainingSample` rows once `TRAINING_MIN_SAMPLES` accumulate, a `TrainingJob` is queued.
- Local Mac trainer claims jobs, builds a custom Ollama model on `mistral:7b` via `Modelfile` few-shot, runs an 80/20 train/test split, and POSTs accuracy back.
- Hand-curated `eval_set.jsonl` (30 cases) plus `compare.py` regression-diff between model versions — _stops silent quality regression as the model evolves_.

## 4. Live Demo / Screenshots

[TBD — no screenshots or demo video committed.] APK builds against `https://app-production-5914.up.railway.app` (Railway deployment, see [frontend/src/config.ts](frontend/src/config.ts)).

## 5. Tech Stack

### Frontend ([frontend/package.json](frontend/package.json))
| Choice | Why |
| --- | --- |
| React Native 0.74.5 (TypeScript) | Single codebase for Android (primary) and iOS scaffolding. |
| Zustand + persist middleware | Lightweight global store with AsyncStorage hydration no Redux boilerplate. |
| React Navigation (native-stack + bottom-tabs) | Standard, performant native stack with conditional gating per permission state. |
| Reanimated + Skia | 60fps charts (`LineChart`, `PieChart`) without JS thread jank. |
| `@react-native-firebase/auth` + Google Sign-In | Production-grade OAuth without rolling our own. |
| `@notifee/react-native` | Local scheduled notifications for SIP reminders, daily/weekly summaries, budget alerts — _no FCM round-trip_. |
| `react-native-keychain` | Refresh token in iOS Keychain / Android Keystore, not AsyncStorage. |
| `js-sha256` | Same hash function client and server use for dedup key compatibility. |

### Backend ([backend/package.json](backend/package.json))
| Choice | Why |
| --- | --- |
| Node.js ≥18 + Express 4 | Familiar HTTP layer with mature middleware ecosystem. |
| Prisma 6 + PostgreSQL | Type-safe queries, first-class migrations, JSON-friendly schema `Decimal(14,2)` for money so no float drift. |
| Zod | Single validation layer reused for both env config and request bodies/params/query. |
| Helmet + CORS + `trust proxy 1` | Security headers + correct client IP behind Railway/DO load balancers. |
| `jsonwebtoken` + Session row | DB-backed refresh token enables real logout + revoke pure JWT can't do that. |
| `firebase-admin` | Verifies Google ID tokens server-side. |
| `morgan` | Structured access logs. |

### Database ([backend/prisma/schema.prisma](backend/prisma/schema.prisma))
PostgreSQL with composite unique constraints, `Decimal(14,2)` for money, and indexes tuned for `(userId, date)`, `(userId, normalizedMerchant)`, `(userId, hash)`. 8 migrations under [backend/prisma/migrations/](backend/prisma/migrations/).

### Infrastructure
| Layer | Choice |
| --- | --- |
| Backend host | Railway (production URL hardcoded in `frontend/src/config.ts`) also configured for DigitalOcean App Platform (`PORT=8080`). |
| DB | Managed Postgres (Railway / DO). |
| Trainer host | Mac (Apple-Silicon recommended) running as a `launchd` LaunchAgent — _zero cloud GPU cost_. |
| LLM runtime | Ollama (`mistral:7b` base + custom `spendy-sms:vN` tag). |

### Third-party APIs
| Service | Purpose |
| --- | --- |
| Firebase Auth | Google Sign-In ID-token verification. |
| BotBiz WhatsApp | OTP delivery (delivery-only we own generation + verification). |
| `otp.dev` | Legacy fallback, kept dormant for fast rollback. |

## 6. Architecture Overview

Three services, one trust boundary. The phone never ships raw OTPs or unparsed sensitive SMS to the backend — the masker (`smsMasker.ts`) and the server-side bank-sender allowlist (`smsSenders.js`) are the gate. The trainer never touches the phone or the public internet beyond the backend it only pulls labelled samples through an authenticated `x-trainer-token` channel.

```
┌─────────────────────────────────┐
│ React Native App (Android/iOS)  │
│  ┌───────────────────────────┐  │
│  │ smsReader → smsValidator  │  │
│  │  → smsMasker → smsParser  │  │
│  │  → smsProcessor (dedup)   │  │
│  └───────────────────────────┘  │
│       │ structured txns         │
│       │ + masked rawSms         │
│       ▼                         │
│  Zustand store (persisted)      │
└─────────┬───────────────────────┘
          │ HTTPS  Bearer JWT
          ▼
┌─────────────────────────────────┐        ┌──────────────────────┐
│ Express API (Node 18)           │◀──────▶│  PostgreSQL          │
│  /auth /api/auth /user          │        │  Prisma schema       │
│  /transactions /categories      │        │  Sessions, Txns,     │
│  /investments /insights         │        │  TrainingSamples,    │
│  /sms /sync-jobs /v1/training   │        │  TrainingJobs, ...   │
└─────────┬───────────────────────┘        └──────────────────────┘
          │ x-trainer-token         x-cron-secret  │
          │ (claim job + samples)   (scheduled)    │
          ▼                                        ▼
┌─────────────────────────────────┐        ┌──────────────────────┐
│ Python Trainer (LaunchAgent)    │        │  Cron / Scheduler    │
│  poll → claim → 80/20 split     │        │  /v1/training/       │
│  greedy-diversify 60 few-shot   │        │     check-queue      │
│  ollama create spendy-sms:vN    │        │     reap-zombies     │
│  validate → POST accuracy       │        └──────────────────────┘
└─────────────────────────────────┘
                │
                ▼
        Ollama (mistral:7b)
```

## 7. Project Structure

```
.
├── backend/                  Node.js + Express API
│   ├── prisma/               Schema + 8 migrations
│   ├── scripts/              DB preflight, backfill, reparse helpers
│   └── src/
│       ├── app.js            Express app composition
│       ├── server.js         HTTP bootstrap + signal handlers
│       ├── config/           env (Zod-validated), Prisma, Firebase
│       ├── middleware/       auth, rateLimiter, validate, errors
│       ├── modules/          Feature slices (auth, transaction, sms, …)
│       ├── routes/           Phone-OTP auth router
│       ├── services/         OTP providers, sessionService
│       └── utils/            normalizers, smsSenders, phone, dateRange
├── frontend/                 React Native 0.74 (TypeScript)
│   ├── App.tsx               Root with theme + gesture root
│   ├── android/, ios/        Native shells
│   └── src/
│       ├── components/       37 reusable UI primitives + charts
│       ├── screens/          Home, Transactions, Analysis, SMS, Auth, …
│       ├── navigation/       Permission-gated RootNavigator + tabs
│       ├── services/         api, smsEngine, smsParser, syncService, …
│       ├── store/            Zustand store (persisted)
│       ├── hooks/            useAppTheme, useSmsSync
│       └── utils/            currency, dates, finance helpers
└── trainer/                  Python Mac-side training service
    ├── train.py              Poll → claim → build Ollama model → report
    ├── eval.py, compare.py   Regression-diff eval against eval_set.jsonl
    ├── eval_set.jsonl        30 hand-curated regression cases
    └── setup.sh              LaunchAgent install/uninstall
```

## 8. Getting Started

### Prerequisites
- Node.js ≥ 18.18, PostgreSQL ≥ 14, Yarn or npm.
- React Native dev environment (Android Studio for the primary target, Xcode for iOS).
- Python 3.10+ and Ollama for the trainer (optional, only needed to retrain).

### Backend
```bash
cd backend
cp .env.example .env          # fill JWT_SECRET, OTP_HMAC_SECRET, FIREBASE_*, BOTBIZ_*, CRON_SECRET, TRAINER_TOKEN
npm install
npx prisma migrate deploy     # or `prisma migrate dev` locally
npm run dev                   # nodemon-style: node --watch src/server.js
```

Env validation runs on startup via Zod ([backend/src/config/env.js](backend/src/config/env.js)) — the server refuses to boot on a missing/invalid value rather than crashing later.

### Frontend
```bash
cd frontend
npm install
# Edit src/config.ts — IS_PRODUCTION=false for local dev
npm start                     # Metro bundler
npm run android               # or `npm run ios`
```

### Trainer (optional)
```bash
cd trainer
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # fill BACKEND_URL, TRAINER_TOKEN
python train.py               # one-shot
bash setup.sh install         # or run as LaunchAgent
```

### Tests
```bash
# Backend (Jest + Supertest, hits a real test DB — see scripts/test-db-preflight.js)
cd backend && npm test
npm run test:unit            # utils only, no DB
npm run test:integration     # module-level
npm run test:coverage

# Frontend (Jest + ts-jest)
cd frontend && npm test

# Trainer eval (regression suite for the trained model)
cd trainer && python eval.py spendy-sms:v1
python compare.py spendy-sms:v1 spendy-sms:v2
```

## 9. API Reference

All routes prefixed with the deployed origin. Auth is `Authorization: Bearer <accessToken>` unless noted.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/health` | Liveness probe. |
| POST | `/api/auth/send-otp` | Send WhatsApp OTP via BotBiz rate-limited. |
| POST | `/api/auth/verify-otp` | Verify OTP, create user (+ default categories) on first login, issue tokens. |
| GET | `/api/auth/profile` | Authed profile fetch. |
| POST | `/api/auth/refresh` | Rotate access token using refresh token. |
| POST | `/api/auth/logout` | Revoke session row. |
| POST | `/auth/google` (alias `/auth/verify-token`) | Verify Firebase ID token and issue session. |
| GET / PUT | `/user/me`, `/user/update` | Profile read/update. |
| GET / POST / PUT / DELETE | `/categories` | Category CRUD (custom + default). |
| GET / POST / PUT / DELETE | `/transactions` | Transaction CRUD. |
| POST | `/transactions/batch` | Batch insert with server dedup. |
| GET / POST | `/mapping` | Merchant → category mappings (with nicknames). |
| GET / POST / PUT / DELETE | `/investments` | Investment CRUD incl. SIP fields. |
| GET | `/insights/category` `?month&year`/`?from&to` | Category breakdown. |
| GET | `/insights/monthly` | Monthly totals. |
| GET | `/insights/trends` | Rolling trend window. |
| POST | `/sms/raw` | Upload masked SMS batch (sender allowlist enforced). |
| GET / POST / PATCH | `/sync-jobs/...` | SyncJob progress reporting. |
| POST | `/v1/training/check-queue` | Cron-only — queue a job if `>= TRAINING_MIN_SAMPLES`. |
| POST | `/v1/training/reap-zombies` | Cron-only — reset/fail jobs missing heartbeats. |
| GET | `/v1/training/job/pending` | Trainer-only — atomic claim. |
| POST | `/v1/training/job/:id/{heartbeat,complete,fail}` | Trainer-only — lifecycle. |
| GET | `/v1/training/status` | Authed — recent jobs. |

## 10. Data Model

Primary entities ([backend/prisma/schema.prisma](backend/prisma/schema.prisma)):

- **User** owns everything via `onDelete: Cascade`. Phone is E.164, optional email, optional `firebaseUid`.
- **Session** — refresh-token row, opaque `token`, `expiresAt`. JWT carries `jti=session.id`.
- **OtpCode** — one row per phone, HMAC-SHA256 `codeHash`, `attempts` cap, single-use `consumed` flag.
- **OTPAttempt** — append-only audit row backing the rate limiter.
- **Transaction** — `Decimal(14,2)` amount, `dedupKey = sha256(amount|YYYY-MM-DD|cardLast4|cleanMerchant)`, unique with `userId`. `categoryId` `SetNull` on category delete.
- **Category** — `(userId, normalizedName, kind)` unique defaults seeded on signup.
- **MerchantMapping** — per-user merchant → category, optional `nickname`.
- **SmsRawMessage** — masked body, `(userId, hash)` unique, links to parsed `Transaction` if any.
- **TrainingSample** — labelled SMS for the trainer, `usedInTraining` flag, optional `trainingJobId`.
- **TrainingJob** — model version counter, `status` state machine (pending → in_progress → completed/failed), heartbeat columns, `retryCount` for the zombie reaper.
- **Investment** — name, type, amount, optional `isSip`/`monthlyAmount`/`nextDate`.
- **SyncJob** — `status`, `total`, `processed`, `errors[]` for client-visible progress.

## 11. Security & Performance Notes

**Trust boundary**
- Bank-sender allowlist enforced server-side in [backend/src/utils/smsSenders.js](backend/src/utils/smsSenders.js) the matching frontend list is a hint, not a gate.
- OTP digits in inbound SMS are masked client-side ([frontend/src/services/smsMasker.ts](frontend/src/services/smsMasker.ts)) before any upload — even if the masked body is stored, no live code is recoverable.
- Server computes the dedup hash itself instead of trusting the client's FNV-1a — a malicious client can't force collisions.

**Auth**
- JWT (15-min default) + opaque refresh in DB `Session` row. Logout = `DELETE FROM Session` → both tokens dead immediately. Older tokens without `jti` are rejected (forced re-login on rollout).
- OTP: HMAC-SHA256 of code with a **separate** `OTP_HMAC_SECRET` (not `JWT_SECRET`), `crypto.timingSafeEqual` compare, `MAX_VERIFY_ATTEMPTS=5`, single-use `consumed`.
- Phone normalized to E.164 before any DB lookup so `+91` and `91` and `0` prefixes can't create duplicate users.

**Rate limiting**
- OTP send: cooldown (default 30s) + hourly cap (default 3) backed by the persistent `OTPAttempt` table — survives process restarts.
- OTP verify: 5 tries per 10-minute rolling window (in-memory). _[Caveat: in-memory, doesn't share across instances — see Gaps.]_
- Concurrent 401s on the client coalesce into a single `/auth/refresh` call ([frontend/src/services/api.ts](frontend/src/services/api.ts)).

**Reliability**
- Training jobs use atomic `updateMany({ where: { id, status: "pending" } })` to claim — two trainers can't race.
- Zombie reaper resets jobs whose `macLastHeartbeat` is older than `ZOMBIE_THRESHOLD_MINUTES` fails permanently after `ZOMBIE_MAX_RETRIES`.
- Backend handles `SIGINT`/`SIGTERM` with Prisma disconnect + graceful HTTP shutdown.
- Helmet enabled, `x-powered-by` disabled, `trust proxy 1` for accurate client IPs behind Railway.

**Performance**
- Indexes on `(userId, date)`, `(userId, normalizedMerchant)`, `(userId, categoryId)`, `(userId, hash)`, `(userId, wasParsed)`.
- Skia-rendered charts run off the JS thread.
- Refresh-token rotation avoids re-querying user on every request.
- 90-day login-merge primes the dedup hash set so the first sync after a re-install doesn't double-write.

## 12. Roadmap (inferred from code/comments)

- **Multi-instance verify limiter** — current 10-minute verify bucket is in-memory move to Redis or DB rows once horizontal scaling kicks in.
- **iOS parity** — iOS scaffolding exists but Android-only flows (`READ_SMS`, `BroadcastReceiver`) need an alternative ingest (manual entry / email parse) for Apple's stricter inbox.
- **Real fine-tuning path** — trainer comment notes the upgrade route: MLX-LM LoRA → merge → GGUF → `ollama create` once Modelfile few-shot stops scaling.
- **`otp.dev` cleanup** — provider has been replaced by BotBiz but the legacy module is still in tree for rollback. Remove once BotBiz has soaked.
- **Public-cron migration** — `/v1/training/check-queue` and `/reap-zombies` rely on `CRON_SECRET` headers an external scheduler (Railway cron, GitHub Actions, etc.) needs to be wired up — see [TBD] above.
- **Investment auto-valuation** — `currentValue` field exists but is user-entered today an integration with a quotes provider would let the trend chart speak to portfolio P&L, not just spend.

## 13. Business Value

**Market.** Every Indian with a bank account gets transaction SMS — that's >900M UPI users. Existing PFM apps are either dead (forced shutdowns following the 2022 RBI account-aggregator clampdown), credential-hungry (a non-starter for a privacy-conscious cohort), or manual (high churn after the novelty week).

**Use case.** A young salaried or self-employed user who lives on UPI, gets paid via direct bank credit, and has zero patience for spreadsheets but high anxiety about month-end balance. Spendly turns "where did all my money go" into a one-tap dashboard with no data handoff.

**Monetization angle.** The architecture is built so the user — not the operator — owns the data. That makes premium tiers (advanced insights, multi-account merge, family sharing, exportable tax-ready statements, investment auto-valuation) the natural revenue surface, instead of monetizing transaction data. A B2B angle exists too: the local-first parser + on-device dedup is a drop-in for any Indian fintech that needs SMS ingest without cloud SMS sync.

**Scalability story.** Backend is stateless Express behind a managed Postgres — horizontal scale is bounded only by DB. Heavy ML lives on the operator's Mac and is fully decoupled (the API doesn't own a GPU) model artefacts are versioned via the `TrainingJob.modelVersion` counter so promotions are reversible. Cost per user at the API tier is dominated by OTP delivery, not compute.

## 14. Engineering Highlights (defensible in a whiteboard)

1. **Server-side dedup key as a SHA-256 of `amount|YYYY-MM-DD|cardLast4|cleanMerchant`** — solves the well-known Postgres "NULL is distinct" footgun in unique constraints by coalescing nullable parts to empty string. Backfill recipe lives in [backend/prisma/migrations/20260425000000_transaction_dedup_key/migration.sql](backend/prisma/migrations/20260425000000_transaction_dedup_key/) and JS recreates it bit-for-bit using `Decimal.toFixed(2)`. _Why it matters: cross-device dedup over re-imports without losing legit same-day same-amount transactions._
2. **Two-token auth with DB-backed refresh.** Pure-JWT logouts are theatre we mint a 15-minute access JWT carrying `jti=sessionId`, store the refresh token as an opaque UUID in the `Session` row, and validate `Session.exists && !expired` on every protected request. Logout = row delete = both tokens revoked.
3. **OTP threat model.** Code is generated server-side, HMAC-SHA256 hashed with a key **distinct** from `JWT_SECRET` (so a JWT_SECRET leak doesn't hand attackers OTP forgery), single-use via a `consumed` flag, capped at 5 verify attempts, and rate-limited at send-time via a persistent `OTPAttempt` table that survives process restarts.
4. **Trainer concurrency.** Multiple Macs could legally exist job claim uses `prisma.trainingJob.updateMany({ where: { id, status: "pending" } })` and treats `count === 0` as "someone else won." A separate cron-driven reaper resets jobs missing heartbeats and permanently fails them after `ZOMBIE_MAX_RETRIES` so a crashed trainer never wedges the queue.
5. **ML eval is a regression suite, not a metric.** `eval.py` runs the model against a hand-curated `eval_set.jsonl` and `compare.py` diffs two model tags — explicitly named "block the release if regressions aren't understood." This catches the case where training accuracy improves but a known edge-case (slice-bank Kotak UPI handle, OTP-as-promo) silently breaks.
6. **Privacy-by-architecture, not policy.** `smsMasker.ts` redacts 4–8 digit groups before upload, the server's `isKnownSender` allowlist drops anything not from a whitelisted bank prefix, and the client's FNV-1a hash is treated as a *hint* — the server recomputes its own SHA-256 so nothing user-controlled can poison the dedup index.
7. **Permission-gated navigator.** The root navigator is one declarative `if/else` ladder over `hydrated → onboardingCompleted → isLoggedIn → smsPermission → notificationPermission → initialSyncDone` ([frontend/src/navigation/RootNavigator.tsx](frontend/src/navigation/RootNavigator.tsx)). Adding a new gating step is one line, not a new flag scattered across screens.
8. **Decimal money everywhere.** `Decimal(14,2)` in Postgres, `js-sha256` shared with the frontend, `formatAmountForKey` coerces to `.toFixed(2)` on both sides — no float drift, no per-locale parse drift.
9. **Retroactive merchant-mapping backfill.** Saving a mapping (or simply categorizing a transaction) runs a kind-filtered `UPDATE transaction SET categoryId = ? WHERE userId AND normalizedMerchant AND categoryId IS NULL AND type = <category.kind>` so older Domino's transactions catch up the moment a single one is tagged. The `categoryId IS NULL` clause never overwrites a manual choice the `kind` filter prevents an EXPENSE mapping bleeding onto an INCOME row. Same helper is reused by both the transaction-create path and the explicit `POST /mapping` path. _Why it matters: matches user intent ("I told the app what Domino's is") without invalidating any prior manual category assignment._
10. **Single-card insight engine.** `buildHomeInsight` returns a 4-state discriminated union (`InsightKind`) rather than four parallel components, so the Home card has one render path and the deep-link target is decided by `kind`. The "is this categorized?" predicate is identical to `getTransactionDisplay`'s — a transaction counts as categorized if `categoryId` is set _or_ a merchant mapping resolves it — so insight counts never desynchronize from what the user sees on the row.
11. **Notification deep-links survive cold start.** Tap intents are written to AsyncStorage by both the foreground (`onForegroundEvent`) and background (`onBackgroundEvent`) handlers, then consumed by `RootNavigator` on every `AppState` resume — not just on mount. So tapping a notification that brings the app from background to foreground (which doesn't remount React) still navigates to the correct snapshot screen. The `snapshotDate` payload makes next-day taps deterministic.

## 15. Contributing

This repo isn't open to external PRs yet. Internal contribution flow:
- Branch off `main`, run `npm run typecheck && npm test` (frontend) and `npm run test:unit && npm run test:integration` (backend) before opening a PR.
- Migrations: `npx prisma migrate dev --name <slug>` on a local DB CI deploys with `prisma migrate deploy`.
- Update `eval_set.jsonl` whenever a real-world SMS the model gets wrong is found — _that_ is how the trainer learns over time.

## 16. License

[TBD — no `LICENSE` file present in the repo.]

## 17. Author / Contact

Maintainer: Prateek Verma. Existing repo-level docs: [README.md](README.md), [README_SHOWCASE.md](README_SHOWCASE.md), [FINANCEFLOW_QA_TEST_PLAN.md](FINANCEFLOW_QA_TEST_PLAN.md), [trainer/README.md](trainer/README.md).

---

## Gaps Found During Analysis _(for the maintainer, not the README)_

The following are real gaps relative to a production-grade pitch. Listed in rough priority order.

1. **No CI/CD.** `.github/` contains only a `java-upgrade/` Dependabot folder — no build/test/deploy workflow. Tests will rot.
2. **`IS_PRODUCTION = true` literal in [frontend/src/config.ts](frontend/src/config.ts).** Production API URL is hardcoded dev builds will hit prod unless someone remembers to flip the constant. Move to `.env` with `react-native-config`.
3. **`GOOGLE_WEB_CLIENT_ID` committed in source.** Not technically a secret (public OAuth client id), but should still come from config so different environments don't clash.
4. **No license file.** Every README badge says "TBD". Pick one before any external collaboration.
5. **No Dockerfile / docker-compose.** Local Postgres bring-up is left to the developer new contributors will struggle.
6. **In-memory verify rate-limiter** ([backend/src/middleware/rateLimiter.js](backend/src/middleware/rateLimiter.js#L61)). Doesn't share across instances — once horizontal scaling lands, an attacker can spread verify attempts across instances. Move to Redis or a DB row pattern like `OTPAttempt`.
7. **iOS not validated.** All ingest paths (`smsReader`, `BroadcastReceiver`) are Android-only the iOS folder builds but the entire SMS pipeline is unreachable. Either gate it explicitly with helpful UX, or invest in an iOS-compatible ingest (manual / email).
8. **Test coverage is uneven.** Backend tests cover auth, transaction (incl. dedup), category, investment, mapping, rateLimiter, sessionService, and utils — but `insights`, `sms`, `syncJob`, `training`, and `user` modules have **no tests**. Frontend has `smsParser`, `smsValidator`, `mappers`, `dedupHash.contract`, `bugfix` — no UI/screen tests at all.
9. **No e2e tests.** Critical flows (OTP login → permissions → first sync → first transaction) aren't exercised end-to-end. Detox would fit.
10. **No observability.** `morgan` is the only telemetry no structured logger, no error reporter (Sentry/Bugsnag), no metrics. The training reaper logs to stdout — fine for a Mac, invisible on Railway without a log drain.
11. **No Dockerfile / health probe spec for managed platforms.** `/health` exists `readiness` (DB ping) does not.
12. **`.env.example` instructs `<REQUIRED>`** but env loader will boot if any required var is missing the prefix-check (Zod catches min-length, but `JWT_SECRET=<REQUIRED>` is 10 chars, **fails**`min(32)`, _good_ — but `OTP_HMAC_SECRET=<REQUIRED>` similarly fails just document this loudly).
13. **Legacy `otp.dev` provider still in tree** ([backend/src/services/otpProvider.js](backend/src/services/otpProvider.js)). Dead code increases audit surface. Delete once BotBiz has soaked enough.
14. **No CHANGELOG / no versioning.** `package.json` says `1.0.0` and `0.0.1` no record of what shipped when.
15. **`FINANCEFLOW_QA_TEST_PLAN.md` is gitignored** but committed. Decide which it should be.
16. **Investment `currentValue` is user-entered.** No quotes integration → trend charts can't show portfolio P&L the feature is half-realized.
17. **No CONTRIBUTING.md / SECURITY.md / CODE_OF_CONDUCT.md.** Standard OSS hygiene if this ever opens up.
18. **`SmsRawMessage` row stores the (masked) body indefinitely.** If retention is part of the privacy pitch, document a TTL job otherwise this contradicts "your bank SMS never leaves your phone."

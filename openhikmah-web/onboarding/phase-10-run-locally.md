# Phase 10 — Run the Application Locally

> **Prerequisites:** Phases [1](./phase-1-what-is-openhikmah.md)–[9](./phase-9-database-and-semantic-search.md)  
> **Evidence tags:** **FACT** · **INFERENCE** · **UNKNOWN**

Phase 9 explained **what Postgres stores**. Phase 10 is **hands-on setup** — how to run OpenHikmah on your machine, which env vars unlock which features, and how to tell success from a misconfiguration.

**This phase documents the workflow.** Run the commands yourself when ready; nothing here mutates application code.

---

## What you are building locally

```mermaid
flowchart LR
  subgraph machine ["Your machine"]
    Bun[bun run dev :3000]
    Browser[Browser]
    LS[(localStorage canvas)]
  end

  subgraph services ["Local services"]
    PG[(Postgres + pgvector)]
    Redis[(Redis — optional)]
  end

  subgraph external ["External APIs"]
    Claude[Anthropic Claude]
    Gemini[Gemini embed + optional LLM]
    QF[Quran Foundation OAuth]
    QC[quran.com search]
    AQ[alquran.cloud fallback]
  end

  Browser --> Bun
  Bun --> PG
  Bun -.-> Redis
  Bun --> Claude
  Bun --> Gemini
  Bun --> QF
  Bun --> QC
  Bun --> AQ
  Browser --> LS
```

**MUST UNDERSTAND NOW:** OpenHikmah is **not** a static frontend. Search, expand, share, auth, and the AI graph all need **server routes + Postgres** for full behavior.

---

## Why Bun?

**FACT:** `package.json` declares `"packageManager": "bun@1"`. CI and the Dockerfile use `oven-sh/setup-bun` / `oven/bun:1-alpine`.

| Reason | Detail |
| --- | --- |
| Official toolchain | `bun install`, `bun run dev`, scripts run via `bun scripts/...` |
| Speed | Fast installs — matters on a large Next.js dependency tree |
| Lockfile | `bun.lock` — CI uses `bun install --frozen-lockfile` |

**FACT:** You do not need Node/npm for day-to-day work if Bun is installed. Playwright e2e still uses `bunx playwright install`.

**UNKNOWN:** Whether maintainers test on a specific Bun patch version beyond `@1` — CI uses `latest`.

---

## Setup walkthrough (ordered steps)

Each step: **what**, **state change**, **success**, **common failure**.

### Step 1 — Clone and install dependencies

```bash
cd openhikmah-web    # your fork clone
bun install
```

| | |
| --- | --- |
| **What** | Installs packages from `bun.lock` into `node_modules/` |
| **State change** | Disk only — no `.env`, no database |
| **Success** | Command exits 0; `node_modules/` populated |
| **Failure** | Missing Bun → install from [bun.sh](https://bun.sh); lockfile conflict → don't delete lockfile without maintainer guidance |

---

### Step 2 — Create `.env.local`

```bash
cp .env.example .env.local
```

| | |
| --- | --- |
| **What** | Next.js loads `.env.local` at dev server startup (not committed) |
| **State change** | New gitignored file |
| **Success** | File exists beside `.env.example` |
| **Failure** | Never commit `.env.local` — contains secrets |

**FACT:** After editing `.env.local`, **restart `bun run dev`**. `NEXT_PUBLIC_*` vars are baked into the client bundle at startup (`.env.example` line 129).

---

### Step 3 — Start Postgres with pgvector

OpenHikmah requires **PostgreSQL 16 + pgvector** (`docker-compose.yml`, migration `0008`).

**Option A — Docker Compose (app + db + redis):**

```bash
# Set DB_PASSWORD in .env or shell, then:
docker compose up -d db
# Optionally: docker compose up -d   # full stack
```

**Option B — Postgres only (matches `.env.example` URL):**

```bash
docker run -d --name openhikmah-db \
  -e POSTGRES_DB=open_hikmah \
  -e POSTGRES_USER=openh \
  -e POSTGRES_PASSWORD=devpassword \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

Set in `.env.local`:

```bash
DATABASE_URL=postgresql://openh:devpassword@localhost:5432/open_hikmah
```

| | |
| --- | --- |
| **What** | Empty Postgres instance with vector extension available |
| **State change** | Docker volume `postgres_data` (compose) or container filesystem |
| **Success** | `docker ps` shows container healthy; port 5432 reachable |
| **Failure** | Port in use → stop other Postgres or change host port mapping; wrong image → must be **pgvector**, not vanilla `postgres:16` |

**INFERENCE:** `.env.example` references a README docker run command that may not appear in the current README — Option B above matches the documented `DATABASE_URL` credentials.

---

### Step 4 — Apply migrations

```bash
bun run db:migrate
bun run db:migrate:concurrent-indexes   # connections locale index — run once locally
```

| | |
| --- | --- |
| **What** | Creates all tables from `lib/infra/db/migrations/` |
| **State change** | Postgres schema (empty tables) |
| **Success** | `Migrations applied` / no error |
| **Failure** | `DATABASE_URL` wrong → connection refused; pgvector missing → migration `0008` fails |

Wrapper: `scripts/migrate.mjs` uses Drizzle migrator. Production Docker also runs `scripts/ensure-tables.mjs` as idempotent bootstrap — optional locally after migrate.

---

### Step 5 — Seed the Quran corpus (required for serious dev)

```bash
bun run scripts/seed-quran.mjs
# or: node scripts/seed-quran.mjs  (scripts are .mjs; package.json has bun wrappers for some)
```

| | |
| --- | --- |
| **What** | Fetches alquran.cloud (Uthmani Arabic + `en.sahih`); upserts ~6236 rows into `verses` |
| **State change** | `verses` table populated |
| **Success** | Log shows merged 6236 verses (warns if count differs) |
| **Failure** | No network → fetch fails; no `DATABASE_URL` → exits immediately |

**FACT:** Without seeding, `getVerses()` misses and `resolveVerse()` **falls back to live alquran.cloud** (`lib/quran/verse-resolver.ts`) — usable for basic verse display but slower and not how CI/integration tests behave.

---

### Step 6 — Seed grounding data (strongly recommended)

```bash
bun run scripts/seed-morphology.mjs   # word_morphology from data/morphology/*.jsonl
bun run embed                         # alias: scripts/embed-corpus.mjs — needs GEMINI_API_KEY
```

| Script | Unlocks |
| --- | --- |
| `seed-morphology.mjs` | **Root** grounded discovery (`discoverCandidates` root path) |
| `embed-corpus.mjs` | **Thematic/contrast** discovery + semantic search + "related by meaning" |

| | |
| --- | --- |
| **What** | Tier-1 grounding tables (Phase 9) |
| **State change** | `word_morphology`, `verse_embeddings` populated |
| **Success** | Morphology logs per file; embed logs `verse_embeddings table now holds N rows` |
| **Failure** | Embed without `GEMINI_API_KEY` → script exits; 429 rate limit → re-run later (idempotent) |

**FACT:** `bun run embed` is defined in `package.json` as `bun scripts/embed-corpus.mjs`.

---

### Step 7 — Set AI keys (minimum for expand)

In `.env.local`, at minimum for **AI connection expand**:

```bash
ANTHROPIC_API_KEY=sk-ant-...
AI_PROVIDER=claude
```

**Cost-saving dev alternative:**

```bash
AI_PROVIDER=gemini
GEMINI_API_KEY=...
```

**FACT:** `AGENTS.md` / `CONTRIBUTING.md`: *"At minimum you need `ANTHROPIC_API_KEY` to test AI connections."* Gemini works when `AI_PROVIDER=gemini`.

**For semantic search / embed corpus:**

```bash
GEMINI_API_KEY=...   # required even when AI_PROVIDER=claude
```

Embeddings are **always Gemini** (`lib/ai/ai.ts`).

---

### Step 8 — Start the dev server

```bash
bun run dev
```

| | |
| --- | --- |
| **What** | `next dev` — Next.js 16 App Router dev server |
| **State change** | Process listening on port 3000 (default) |
| **Success** | Terminal shows ready; [http://localhost:3000](http://localhost:3000) loads |
| **Failure** | Port 3000 busy → `bun run dev -- -p 3001` and update `NEXT_PUBLIC_APP_URL`; missing deps → re-run `bun install` |

**FACT:** Set `NEXT_PUBLIC_APP_URL=http://localhost:3000` in `.env.local` for OAuth redirect consistency.

---

## Environment variable tiers

### Tier 0 — App shell only

| Variable | Required? |
| --- | --- |
| `DATABASE_URL` | **Yes** for DB-backed features (expand, share, connections cache) |
| `NEXT_PUBLIC_APP_URL` | Recommended |

**Works without AI keys:** Static/marketing pages, canvas UI shell, localStorage canvas (client-only).

**INFERENCE:** Verse fetch may work via alquran.cloud fallback if corpus not seeded — but expand/share/graph routes hit Postgres and fail if DB is down.

### Tier 1 — Core product (contributor default)

| Variable | Enables |
| --- | --- |
| `ANTHROPIC_API_KEY` (+ `AI_PROVIDER=claude`) | AI expand, connection reasons |
| `GEMINI_API_KEY` | Query embedding for semantic search; required to run `embed-corpus` |
| Seeded `verses` + migrations | Fast local corpus, validation parity with CI |

### Tier 2 — Auth + social + workspaces

| Variable | Enables |
| --- | --- |
| `NEXT_PUBLIC_QF_CLIENT_ID` | Browser OAuth client id |
| `QF_CLIENT_SECRET` | Server token exchange |
| `QF_AUTH_BASE` / `NEXT_PUBLIC_QF_AUTH_BASE` | Prelive: `https://prelive-oauth2.quran.foundation` |
| Redirect URI registered | `http://localhost:3000/callback` with Quran Foundation |

**FACT:** `.env.example` documents **prelive** QF for local dev; production OAuth has stricter feature enablement.

### Tier 2b — Dev auth bypass (no QF OAuth yet)

When QF redirect is not registered:

```bash
DEV_AUTH_TOKEN=<long-random-secret>
DEV_AUTH_QF_ID=dev-admin
DEV_AUTH_USERNAME=devadmin   # optional
ADMIN_QF_IDS=dev-admin       # if testing /admin
```

In browser console (dev only):

```javascript
await window.__devLogin('<DEV_AUTH_TOKEN>')
```

**FACT:** Hard-disabled when `NODE_ENV=production` (`lib/auth/social-auth.ts`, `components/providers.tsx`).

### Tier 3 — Optional accelerators

| Variable | Purpose |
| --- | --- |
| `REDIS_URL` | Query embedding cache, rate limiter, token cache — app works without it |
| `GEMINI_API1`…`GEMINI_API5` | Admin backfill loop only — not live traffic |
| `ADMIN_QF_IDS` | `/admin` console access |

---

## Feature × requirements matrix

| Feature | Postgres | Corpus seed | Embeddings | AI key | Auth |
| --- | --- | --- | --- | --- | --- |
| Home / marketing | No | No | No | No | No |
| Canvas + localStorage | No* | No* | No | No | No |
| Search by ref `2:255` | No* | Prefer yes | No | No | No |
| Keyword search | No | Prefer yes | No | No | No |
| Related by meaning | **Yes** | **Yes** | **Yes** | Gemini embed | No |
| Expand (Theme/Root/Contrast) | **Yes** | **Yes** | Root/embed per kind | **Yes** | No |
| Share canvas URL | **Yes** | No | No | No | No |
| Bookmarks sync | **Yes** | No | No | No | QF OAuth or dev auth |
| Saved workspaces | **Yes** | No | No | No | Auth |
| Admin panel | **Yes** | Varies | Varies | Varies | `ADMIN_QF_IDS` |

\* Canvas and ref search can partially work via client state + live alquran.cloud fallback, but this is not representative of production or CI.

---

## Verify your setup (smoke checklist)

Run after Steps 1–8:

| # | Check | How | Pass criteria |
| --- | --- | --- | --- |
| 1 | Health | `curl -s http://localhost:3000/api/health` | `{"status":"ok"}` |
| 2 | Home | Open `/` | Page loads, no 500 |
| 3 | Canvas | Open `/canvas` | Empty state or restored localStorage graph |
| 4 | Verse API | `curl -s http://localhost:3000/api/verse/2/255` | JSON with `ref`, `arabicText`, `translation` |
| 5 | Search | ⌘K → search `2:255` or `mercy` | Results appear |
| 6 | Expand | Add `2:255` → expand Theme | New nodes + edges (needs Tier 1) |
| 7 | Semantic | Search with concept phrase; check "related" | Needs embeddings + `GEMINI_API_KEY` |
| 8 | Share | Toolbar Share (with nodes on canvas) | URL copied; opens in new tab (needs Postgres) |

**FACT:** `/api/health` returns static OK — it does **not** verify Postgres connectivity.

**INFERENCE:** A passing health check + failing expand usually means **DB down**, **missing seed**, or **missing AI key** — not a broken Next.js install.

---

## Redis (optional)

**FACT:** `.env.example`: leave `REDIS_URL` unset → in-process / Postgres fallbacks (`lib/infra/redis.ts`).

**FACT:** `docker compose up` starts Redis and wires `REDIS_URL` automatically.

**USEFUL LATER:** Redis reduces repeated Gemini embed calls for popular search queries (7-day cache in `semantic-search.ts`).

---

## Running tests locally (not Phase 10 depth, but you'll need them before PR)

| Command | Needs | Notes |
| --- | --- | --- |
| `bun run test` / `test:ci` | Nothing external | Unit tests; mock DB |
| `bun run test:integration` | **Docker running** | Testcontainers spins pgvector Postgres |
| `bun run test:e2e` | Docker Postgres + migrate + `DEV_AUTH_*` in `.env.local` | Playwright on port **3100** |
| `bun run lint` / `typecheck` / `format:check` | Nothing | CI parity |

**FACT:** `.husky/pre-push` runs integration tests — **`docker ps` must succeed before `git push`** (AGENTS.md).

**FACT:** E2E CI runs `db:migrate` + `db:migrate:concurrent-indexes` but does **not** run full corpus seed — e2e tests use dev auth + live/fallback paths appropriate to CI env.

---

## Common failures (diagnosis first)

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `connection refused` on expand/share | Postgres not running or wrong `DATABASE_URL` | Start pgvector container; verify URL |
| Expand returns error toast immediately | Missing `ANTHROPIC_API_KEY` / wrong provider | Set Tier 1 keys; restart dev server |
| Expand succeeds but always legacy/slow | No morphology/embeddings seeded | Run Phase 9 seed scripts |
| "Related by meaning" never appears | No `verse_embeddings` or no `GEMINI_API_KEY` | `bun run embed` + key |
| OAuth redirect error | `NEXT_PUBLIC_APP_URL` mismatch or callback not registered | Use `http://localhost:3000`; register callback with QF |
| Sign-in works but no admin | User not in `ADMIN_QF_IDS` | Add your `DEV_AUTH_QF_ID` or QF `sub` |
| `git push` hook fails mysteriously | Docker not running for Testcontainers | Start Docker Desktop |
| Changed env, behavior unchanged | Dev server not restarted | Kill and `bun run dev` again |
| Semantic search works once then fast | Redis caching — expected | Not an error |

**MUST UNDERSTAND NOW:** Do not invent workarounds (skip hooks, mock production DB) before checking the table above — this repo's failures are usually **missing service or missing seed**, not mysterious Next.js bugs.

---

## Minimal `.env.local` starter (copy and fill)

```bash
# ─── Required for DB-backed dev ───
DATABASE_URL=postgresql://openh:devpassword@localhost:5432/open_hikmah
NEXT_PUBLIC_APP_URL=http://localhost:3000

# ─── AI expand ───
ANTHROPIC_API_KEY=
AI_PROVIDER=claude

# ─── Semantic search + embed-corpus ───
GEMINI_API_KEY=

# ─── Dev auth (optional — skip QF OAuth early) ───
# DEV_AUTH_TOKEN=
# DEV_AUTH_QF_ID=dev-admin
# ADMIN_QF_IDS=dev-admin

# ─── QF OAuth (optional until testing bookmarks/sync) ───
# NEXT_PUBLIC_QF_CLIENT_ID=
# QF_CLIENT_SECRET=
# QF_AUTH_BASE=https://prelive-oauth2.quran.foundation
# NEXT_PUBLIC_QF_AUTH_BASE=https://prelive-oauth2.quran.foundation
```

---

## Full first-time command sequence (copy-paste)

Assumes Docker installed, Bun installed, `.env.local` filled with AI keys:

```bash
bun install
cp .env.example .env.local
# edit .env.local

docker run -d --name openhikmah-db \
  -e POSTGRES_DB=open_hikmah \
  -e POSTGRES_USER=openh \
  -e POSTGRES_PASSWORD=devpassword \
  -p 5432:5432 \
  pgvector/pgvector:pg16

bun run db:migrate
bun run db:migrate:concurrent-indexes
bun scripts/seed-quran.mjs
bun scripts/seed-morphology.mjs
bun run embed

bun run dev
# open http://localhost:3000/canvas
```

---

## MUST UNDERSTAND NOW

1. **Bun** is the project's package manager and script runner.
2. **Postgres + pgvector** is required for expand, share, graph cache, and semantic ranking — not optional for core product dev.
3. **Seed order:** migrate → quran → morphology → embed (Phase 9).
4. **`ANTHROPIC_API_KEY`** (or Gemini provider) for expand; **`GEMINI_API_KEY`** additionally for semantic/embed paths.
5. **Restart dev server** after `.env.local` changes, especially `NEXT_PUBLIC_*`.
6. **Docker** required for `test:integration` and pre-push hook — not necessarily for `bun run dev` itself.

---

## USEFUL LATER

- `docker compose up` — full production-like stack (app + db + redis).
- `bun run prewarm` — offline graph warming against running app.
- `scripts/seed-translations.mjs` — non-English edition rows.
- Playwright: `bun run test:e2e` on port 3100 with `DEV_AUTH_*` set.

---

## IGNORE FOR NOW

- Production deploy (`Dockerfile`, `scripts/deploy.sh`) — unless you're doing infra contributions.
- `GEMINI_API1`–`5` pool — admin backfill loop only.
- Tuning Postgres/HNSW parameters — defaults work for local dev.

---

## Phase 10 checkpoint questions

1. Which three seed scripts should you run after migrate for full grounded expand + semantic search?
2. Why is `GEMINI_API_KEY` required even when `AI_PROVIDER=claude`?
3. What URL and env vars does QF OAuth need for local sign-in?
4. How do you bypass OAuth in dev, and why does it not work in production builds?
5. What command proves the Next.js process is up — and what does it *not* prove?

---

**Next:** [Phase 11 — Browser ↔ code correlation](./phase-11-browser-code-correlation.md) — perform one runtime flow in the browser and map Network tab → API → library → database.

Say **“continue to Phase 11”** when you have run through this checklist (or hit a specific failure you want to debug), or ask to walk through any Step in more detail.

# Phase 7 — Grounded AI Connections (Deep Dive)

> **Prerequisites:** Phases [1](./phase-1-what-is-openhikmah.md)–[6](./phase-6-runtime-walkthroughs.md)  
> **Evidence tags:** **FACT** · **INFERENCE** · **UNKNOWN**

Phase 6 traced expand-to-canvas at a summary level. Phase 7 is the **trust-model deep dive** — the pipeline that makes OpenHikmah “grounded” instead of a free-form Quran chatbot.

**Central sentence (memorize):**

> **Data discovers; AI articulates.**

The model may **select** among real candidate verses and **explain** why — it may **not** invent verse references on the preferred path.

---

## The separation of powers

| Role | Module | What it does | Probabilistic? |
| --- | --- | --- | --- |
| **Discover** | `lib/ai/connection-discovery.ts` | Returns up to 12 real `surah:ayah` refs from Postgres | **No** — SQL / pgvector |
| **Articulate** | `lib/ai/connection-generator.ts` → `generateGroundedConnections` | LLM picks ≤3 from candidates + writes reasons | **Yes** — but refs gated |
| **Orchestrate + cache** | `lib/ai/graph-service.ts` | Cache hit → skip AI; miss → discover → generate → persist | Mixed |
| **Legacy fallback** | `generateConnections` | LLM proposes refs from memory; `isValidRef` + `getVerses` filter | **Yes** — only when grounding data missing |

**FACT:** `connection-generator.ts` header states it is *“The ONLY module that calls the AI”* for connections.

**FACT:** `graph-service.ts` header: *“Reads connections from Postgres; only on a miss does it call the AI, then writes the result back so every later reader gets it for free.”*

---

## End-to-end pipeline (one diagram)

```mermaid
flowchart TB
  subgraph client ["Browser"]
    VN[VerseNode.handleExpandSelect]
    PE[setPendingExpand]
    RE[HikmahCanvas.runExpansion]
    GER[getExpansionRefs]
    VN --> PE --> RE
    RE --> GER
  end

  subgraph api ["API boundary"]
    RT[POST /api/connections]
    IV[isValidRef + field caps]
    RT --> IV
  end

  subgraph graph ["lib/ai/graph-service.ts"]
    GC[getConnections]
    RA[readActiveRows — cache read]
    SF[singleFlight dedupe]
    GFC[generateConnectionsForCell]
    GLC[generateLocalizedCell — non-en]
    HY[hydrate → resolveVerse]
    GC --> RA
    RA -->|hit| HY
    RA -->|miss| SF
    SF --> GFC
    SF --> GLC
  end

  subgraph discover ["Data discovers"]
    DC[discoverCandidates]
    RC[rootCandidates — word_morphology]
    SC[semanticCandidates — verse_embeddings]
    DC --> RC
    DC --> SC
  end

  subgraph ai ["AI articulates"]
    GGC[generateGroundedConnections]
    GC2[generateConnections — legacy]
    PAR[parseRawConnections]
    ALW["allowed.has(c.ref)"]
    GGC --> PAR --> ALW
  end

  subgraph persist ["Postgres"]
    CONN[(connections)]
    AIG[(ai_generations)]
  end

  RE -->|excludeRefs| RT
  RT --> GC
  GFC --> DC
  DC -->|candidates.length > 0| GGC
  DC -->|candidates empty + first request| GC2
  DC -->|candidates empty + get more| EMPTY[return []]
  GGC --> CONN
  GC2 --> CONN
  GGC --> AIG
  GC2 --> AIG
  CONN --> HY
  HY --> RE
```

---

## The twelve pipeline questions (answered)

The original onboarding spec asked these in order. Use this as a checklist; Steps 0–9 below expand each answer.

| # | Question | Answer (grounded path) |
| --- | --- | --- |
| 1 | What starts the request? | User selects Theme/Root/Contrast → `VerseNode` → `setPendingExpand` → `HikmahCanvas.runExpansion` → `POST /api/connections` |
| 2 | How are candidate verses determined? | `discoverCandidates(fromRef, kind, 12, excludeRefs)` — SQL over `word_morphology` (root) or pgvector neighbors (thematic/contrast) |
| 3 | Which parts use Arabic roots? | **`kind === "root"` only** → `rootCandidates()` in `connection-discovery.ts` |
| 4 | Which parts use embeddings? | **`thematic` and `contrast`** → `semanticCandidates()` → `similarVerses()` → `verse_embeddings` + cosine distance |
| 5 | Where does Claude/Gemini enter? | Only on cache miss, inside `generateGroundedConnections` or legacy `generateConnections` → `callAIDetailed(..., { feature: "connections" })` → `resolveProvider("connections")` picks Claude (default) or Gemini per admin flags / env (`lib/ai/ai.ts`) |
| 6 | What context does the model receive? | Source ref + Arabic + translation + numbered candidate list (`- ref — translation`) + kind-specific selection task + Maturidi/Hanafi framing + appended Tanzih |
| 7 | What schema/format is expected? | JSON array only: `[{ "ref": "surah:ayah", "reason": "one sentence" }, ...]` — at most 3 entries; no markdown wrapper |
| 8 | How is output parsed? | `parseRawConnections(text)` — regex extracts first `[...]`, `JSON.parse`, filters objects with non-empty `ref` + `reason` strings |
| 9 | How are verse references checked? | **Grounded:** `allowed.has(c.ref)` where `allowed` = discovery set. **Legacy:** `isValidRef` + `getVerses` corpus lookup. Both paths drop `fromRef`. |
| 10 | What happens when validation fails? | Malformed/unparseable → `ConnectionParseError` → API **502** (retry). Valid parse but no surviving refs → `[]` (200). Rate limit → **429**. |
| 11 | What theological constraints are added? | `tanzihDirective()` appends `TANZIH_CONSTRAINT`; scholar prompt frames Maturidi/Hanafi tradition — neither is removable via admin prompt overrides |
| 12 | What becomes a canvas edge? | `ConnectionResult[]` returned → `runExpansion` → `addVerseNode` + `buildConnectionEdge` + `addConnectionEdge` — edge carries `kind` + `reason` in `edge.data` |

**FACT:** Embeddings for discovery use Gemini (`embed()` in `semantic-search.ts`); connection **text generation** uses Claude or Gemini via `callAIDetailed`. These are separate call sites.

**UNKNOWN:** Which provider runs in production without inspecting deployed env/admin flags — code default is Claude when unset.

---

## Slow narrative (read once, then trace in code)

Imagine you expand **2:255** by **Theme**.

You already have the verse on the canvas — its Arabic and translation travel with the request. The client also sends every thematic target ref already connected from this node (`excludeRefs`), so “get more” cannot repeat the same three ayahs.

The API validates the ref shape and hands off to `getConnections`. Postgres is consulted first: if this `(2:255, thematic, locale)` cell was generated before and those targets are not excluded, you get cached rows — **no LLM bill**.

On a miss, the server discovers up to twelve real neighbors from embedding similarity — deterministic math, not model memory. Those refs hydrate from the local Quran corpus. Only then does the model run: it sees the source verse, the candidate list, and instructions to pick the best three thematic links and explain each in one sentence — within Maturidi/Hanafi framing and Tanzih.

The model's JSON comes back. Code extracts the array, drops any ref not in the twelve, drops self-links, caps at three, and persists survivors to `connections`. The response hydrates full verse cards plus reasons. Back in the browser, nodes fan out radially; edges appear with explanation pills you can click in the sidebar.

If the model refuses or returns garbage JSON, you get a 502 — not a silent empty canvas. If the grounded pool is genuinely exhausted on “get more,” you get an empty array and an informational “no more connections” notice. That distinction protects both theology and UX.

---

## Step 0 — What triggers the pipeline

**FACT:** User picks Theme / Root / Contrast in `ExpandMenu` → `VerseNode.handleExpandSelect` → `setPendingExpand({ nodeId, ref, kind })`.

**FACT:** `HikmahCanvas` `useEffect` on `pendingExpand` clears pending state and calls `runExpansion` with the source node's `arabicText`, `translation`, and position.

**FACT:** First canvas node auto-expands **thematic** via `pendingAutoExpand` — same `runExpansion` path.

---

## Step 1 — Client builds `excludeRefs` (“get more”)

| | |
| --- | --- |
| **Where** | `store/canvas.ts` → `getExpansionRefs(nodeId, kind)` |
| **Algorithm** | Edges where `e.source === nodeId && e.data.kind === kind` → target node refs |
| **Why** | Repeat expand must surface **new** connections, not re-serve the same targets |

**FACT:** `runExpansion` sends `{ fromRef, kind, arabicText, translation, excludeRefs }` to `POST /api/connections`.

**INFERENCE:** `excludeRefs` is a **client-side session list** (what is already on the canvas for this kind), not the full Postgres graph history. A verse connected on the canvas is excluded from the next “get more” even if it came from a different expansion session.

---

## Step 2 — API boundary (system boundary validation)

**Where:** `app/api/connections/route.ts`

| Check | Purpose |
| --- | --- |
| Required fields | `fromRef`, `kind`, `arabicText`, `translation` |
| `kind ∈ { thematic, root, contrast }` | Rejects unknown edge kinds |
| `isValidRef(fromRef)` | Canonical `"surah:ayah"` within real Hafs bounds |
| `excludeRefs` array, each `isValidRef`, max 100 | Prevents unbounded prompt injection |
| Text length ≤ 5000 | Bounds AI prompt size |

**FACT:** On success, calls `getConnections(..., { clientKey, excludeRefs, locale })` where `locale = await getUiLocale()`.

**FACT:** Error mapping:

| Error | HTTP | Client behavior |
| --- | --- | --- |
| `RateLimitError` | 429 | `connectionsFailed` notice |
| `ConnectionParseError` | 502 | Retry-worthy — **not** “no connections” |
| Empty `[]` | 200 | Client distinguishes exhausted vs first-time empty |

**MUST UNDERSTAND NOW:** An empty JSON array and a 502 parse failure are **different semantics**. Empty = well-formed “nothing to show.” 502 = transient upstream failure — must **not** be cached as exhaustion.

---

## Step 3 — `getConnections`: cache-first orchestration

**Where:** `lib/ai/graph-service.ts` → `getConnections`

### 3a — Cache read

**FACT:** `readActiveRows(fromRef, kind, locale, excludeRefs)` selects from `connections` where:

- `from_ref`, `kind`, `locale` match
- `status = 'active'`
- `to_ref NOT IN excludeRefs` (when provided)
- ordered by `toRef`, limit 200

**FACT:** For non-`en` locales, a cache hit requires **every** active English row to have a matching locale row (ref-by-ref completeness gate). Partial locale sets re-enter the miss path.

**FACT:** On hit → `hydrate(rows, kind)` → `resolveVerse(toRef)` for each row → `ConnectionResult[]`. **No AI.**

### 3b — Cache miss guards

| Guard | Purpose |
| --- | --- |
| `consume(`gen:${clientKey}`)` | Rate-limits expensive miss path per client IP |
| `singleFlight(key, factory)` | Concurrent identical misses share one generation |
| `resolveProvider` / `resolveModel` | Provider+model resolved once, threaded through key + persist |

**FACT:** `cellKey` = `` `${fromRef}:${kind}:${locale}:${provider}:${model}:${sorted excludeRefs}` ``

**FACT:** Non-`en` miss → `generateLocalizedCell`: ensures English rows exist, then `translateReason` per missing row (separate budget consumption). English **selection** is always canonical; locale only affects **reason text**.

---

## Step 4 — `generateConnectionsForCell`: the fork

**Where:** `graph-service.ts` → `generateConnectionsForCell` (exported for admin backfill too)

```typescript
const candidates = await discoverCandidates(fromRef, kind, undefined, excludeRefs);
const calledAI = candidates.length > 0 || excludeRefs.length === 0;

const generated =
  candidates.length > 0
    ? await generateGroundedConnections(...)
    : excludeRefs.length > 0
      ? []
      : await generateConnections(...);  // legacy
```

### Decision table

| `candidates` | `excludeRefs` | Path | AI called? | Typical meaning |
| --- | --- | --- | --- | --- |
| > 0 | any | **Grounded** | Yes | Normal case |
| 0 | > 0 | **Return []** | No (`calledAI` still true*) | “Get more” exhausted grounded pool |
| 0 | 0 | **Legacy** | Yes | Morphology/embeddings not seeded for verse |

\* **FACT:** `calledAI = candidates.length > 0 || excludeRefs.length === 0` — empty pool on “get more” still marks `calledAI: true` so admin backfill can record exhaustion correctly.

**MUST UNDERSTAND NOW:** Legacy path **never runs** when `excludeRefs.length > 0`. Without this rule, “get more” would regenerate similar refs from model memory and defeat pagination.

---

## Step 5 — `discoverCandidates`: deterministic discovery

**Where:** `lib/ai/connection-discovery.ts`

**FACT:** Default limit = **12** candidates per generation pass.

### By kind

| Kind | Source table | Algorithm |
| --- | --- | --- |
| `root` | `word_morphology` | Distinct roots for source ref → verses sharing those roots → rank by `count(distinct root)` DESC |
| `thematic` | `verse_embeddings` | `semanticCandidates` → pgvector nearest neighbors |
| `contrast` | `verse_embeddings` | **Same neighbor pool as thematic** — AI selects genuinely opposing ones |

**FACT:** `semanticCandidates` (`lib/quran/semantic-search.ts`): loads source embedding → `nearest()` cosine similarity → returns refs. Returns `[]` if source has **no stored embedding**.

**FACT:** `excludeRefs` passed through to SQL (`notInArray`) for both root and semantic paths.

**FACT:** Empty `[]` from discovery means *“no grounding data for this verse”* — **not** an error. Callers decide fallback vs exhaustion.

### Seeding dependencies (offline)

| Data | Script | Needed for |
| --- | --- | --- |
| `verses` | `scripts/seed-quran.mjs` | All hydration |
| `word_morphology` | `scripts/seed-morphology.mjs` | `root` discovery |
| `verse_embeddings` | `scripts/embed-corpus.mjs` | `thematic` / `contrast` discovery |

**INFERENCE:** A fresh local dev DB with Quran seeded but **no** embeddings/morphology will hit **legacy** `generateConnections` on first expand — until grounding tables are populated.

---

## Step 6 — `generateGroundedConnections`: AI articulates (preferred path)

**Where:** `lib/ai/connection-generator.ts`

### 6a — Hydrate candidates before prompting

```typescript
const verseMap = await getVerses(candidateRefs);
const candidates = candidateRefs
  .map((ref) => verseMap.get(ref))
  .filter((v) => v !== undefined && v.ref !== fromRef);
if (candidates.length === 0) return [];  // no LLM call
```

**FACT:** If no candidate resolves from local corpus, returns `[]` **without calling the AI**.

### 6b — Prompt construction

| Piece | Source |
| --- | --- |
| Base template | `getPrompt("connection.selection", SELECTION_FALLBACK_TEMPLATE)` |
| Task wording | `KIND_SELECTION[kind]` — differs per kind |
| Candidate list | `- {ref} — {translation}` per verse |
| Maturidi/Hanafi framing | In template |
| Rule: choose ONLY from list | In template |
| **Tanzih** | `tanzihDirective()` — **appended, non-overridable** |
| Locale | `languageDirective(locale)` — appended for non-`en` |

**FACT:** `tanzihDirective()` appends `TANZIH_CONSTRAINT` from `lib/ai/theological-constraints.ts`:

> *“strict Tanzih (divine transcendence): never describe or imply physical form, spatial location, or resemblance to created things (Tashbih)”*

**FACT:** Tanzih and language directives are **not** `{{placeholders}}` in admin-overridable templates — they are appended after `renderTemplate` so DB prompt overrides cannot omit them (see Phase 3).

### 6c — LLM call + audit log

**FACT:** `callAIDetailed(prompt, { feature: "connections", provider, model, ... })`

**FACT:** Best-effort insert into `ai_generations` (fromRef, kind, model, tokens, promptVersion). Logging failure does **not** fail generation.

### 6d — Parse + validate (the anti-hallucination gate)

```typescript
const allowed = new Set<string>(candidates.map((v) => v.ref));
const chosen = parseRawConnections(text)
  .filter((c) => allowed.has(c.ref) && c.ref !== fromRef)
  .slice(0, 3);
```

| Step | Deterministic rule |
| --- | --- |
| `parseRawConnections` | Extract first `[...]` JSON array; validate shape; throw `ConnectionParseError` if malformed |
| `allowed.has(c.ref)` | **Drop any ref not in discovery set** — even if model “knows” a valid verse |
| `c.ref !== fromRef` | Never connect verse to itself |
| `.slice(0, 3)` | Cap at 3 edges per generation |
| Map through `verseMap` | Hydrate full `ConnectionResult` |

**FACT:** Test proves model returning `9:99` when not in candidate list → dropped (`connection-generator.test.ts`: *“rejects any ref the model returns that was not in the candidate set”*).

**FACT:** Well-formed `[]` from model → valid empty selection (no throw). Malformed prose / refusal → `ConnectionParseError` → API 502.

**MUST UNDERSTAND NOW:** Grounded validation is **`allowed.has(c.ref)`**, not merely `isValidRef`. A fabricated but syntactically valid ref is rejected if discovery did not surface it.

---

## Step 7 — `generateConnections`: legacy fallback

**When:** `discoverCandidates` returns `[]` **and** `excludeRefs.length === 0` (first-time miss, no grounding data).

**FACT:** Uses `getPrompt("connection.legacy", LEGACY_FALLBACK_TEMPLATE)` + `KIND_INSTRUCTIONS[kind]`.

**FACT:** Model may propose any refs — filtered by:

1. `parseRawConnections`
2. `isValidRef(c.ref) && c.ref !== fromRef`
3. `getVerses(candidates.map(c => c.ref))` — **must exist in local corpus**
4. `.slice(0, 3)`

**FACT:** Refs that pass `isValidRef` but miss corpus lookup are silently dropped — no external API hydration on this path.

**INFERENCE:** Legacy is a **bootstrap path** for unseeded grounding data, not the long-term product behavior. Production with full seeds should almost always hit grounded path.

---

## Step 8 — Persistence and race handling

**Where:** end of `generateConnectionsForCell`

**FACT:** Insert into `connections`:

```typescript
{ fromRef, toRef: g.ref, kind, reason: g.reason, model, locale: "en" }
.onConflictDoNothing()
```

**FACT:** Unique index on `(from_ref, to_ref, kind, locale)` — concurrent generations on two instances: loser re-reads winner's persisted reason.

**FACT:** Persist failure is logged + `incr("gen_persist_failed")` — caller still returns generated results **this request**, but next request pays AI cost again.

**FACT:** `connections.status` can be `active | flagged | retired` — admin review queue at `/admin/connections`. Canvas reads **active** rows only.

---

## Step 9 — Response back to canvas

**FACT:** `hydrate` / mapped `ConnectionResult[]` includes: ref, arabicText, translation, surah names, **reason**, **kind**.

**FACT:** `runExpansion` loop:

- Empty array + `excludeRefs.length > 0` → `ExhaustedExpansionError` → “no more connections”
- Empty array + first request → generic “no connections found”
- Non-empty → staggered node placement (`radialPos`, `findFreeSlot`), `addVerseNode`, `addConnectionEdge`

**FACT:** If target ref already on canvas → draw edge only (no duplicate node).

---

## Three edge kinds — same pipeline, different discovery + selection intent

| Kind | Discovery question | AI selection question (`KIND_SELECTION`) |
| --- | --- | --- |
| **thematic** | Who is semantically near? (embeddings) | Pick 3 that **share theological theme** |
| **root** | Who shares Arabic roots? (morphology) | Pick 3 where shared root **carries relevant meaning** |
| **contrast** | Who is semantically near? (same pool as thematic) | Pick 3 with **clearest opposing concept** |

**INFERENCE:** Contrast is the subtlest kind — discovery is **similarity-based**, articulation is **opposition-based**. The model must do theological work in selection, not discovery.

---

## Deterministic vs probabilistic — contributor checklist

| Step | Deterministic? | Can you “fix a test” by loosening? |
| --- | --- | --- |
| `isValidRef` | Yes | **Never** |
| `discoverCandidates` SQL | Yes | Fix seed data, not bounds |
| `allowed.has(c.ref)` | Yes | **Never** |
| `getVerses` hydration | Yes | Fix corpus, not skip lookup |
| `parseRawConnections` | Yes | Fix prompt/model, not parser |
| `tanzihDirective` append | Yes | **Never** remove |
| Reason wording | No (LLM) | Admin review + prompt changes, with PR disclosure |
| Which 3 of 12 candidates | No (LLM) | Accept variance within validation |

---

## Failure modes you will see in development

| Symptom | Likely cause | Where to look |
| --- | --- | --- |
| First expand → empty, no error | No embeddings/morphology seeded; legacy returned nothing valid | Run seed scripts; check `discoverCandidates` return |
| “Get more” → “no more connections” | Grounded pool exhausted for kind | Expected — `excludeRefs` + empty candidates |
| Expand → error toast, 502 in network | `ConnectionParseError` — refusal/truncated JSON | Model response; retry |
| 429 on expand | Rate limit on miss path | `lib/infra/rate-limit.ts`; cache hits unaffected |
| Same connections every time | Cache hit — working as designed | `connections` table |
| Turkish UI, English reasons briefly | Locale translation in progress / budget exhausted | `generateLocalizedCell` partial serve |

---

## Key tests (read these before changing the pipeline)

| File | What it locks |
| --- | --- |
| `__tests__/lib/ai/connection-generator.test.ts` | Parse errors, `allowed.has`, corpus hydration, Tanzih/locale in prompts |
| `__tests__/lib/ai/connection-discovery.test.ts` | Root ranking, excludeRefs in SQL, kind routing |
| `__tests__/lib/ai/graph-service.test.ts` | Cache hit/miss, single-flight, excludeRefs, locale completeness |
| `__tests__/integration/graph.integration.test.ts` | End-to-end with real Postgres |
| `__tests__/components/canvas/HikmahCanvas.test.tsx` | Client empty vs exhausted behavior |

---

## Files to keep open while debugging expand

| File | Role |
| --- | --- |
| `components/canvas/HikmahCanvas.tsx` | `runExpansion`, client error semantics |
| `store/canvas.ts` | `getExpansionRefs` |
| `app/api/connections/route.ts` | HTTP boundary + error mapping |
| `lib/ai/graph-service.ts` | Cache, single-flight, locale, persist |
| `lib/ai/connection-discovery.ts` | Candidate SQL |
| `lib/quran/semantic-search.ts` | Embedding neighbors |
| `lib/ai/connection-generator.ts` | Prompts, parse, grounded vs legacy |
| `lib/quran/quran-corpus.ts` | `isValidRef`, `getVerses` |
| `lib/ai/theological-constraints.ts` | `TANZIH_CONSTRAINT` |
| `lib/infra/db/schema.ts` | `connections`, `ai_generations`, indexes |

---

## MUST UNDERSTAND NOW

1. **Grounded path:** discovery produces refs → AI selects subset → `allowed.has(c.ref)` enforces subset.
2. **Legacy path:** only when discovery empty **and** not a “get more” request.
3. **Empty `[]` ≠ parse failure** — different HTTP status, different client UX, different admin backfill semantics.
4. **English selection is canonical** — non-`en` translates reasons, not re-picks verses.
5. **Postgres `connections` is the product graph** — canvas is session layout; sharing saves canvas JSON, not this table.
6. **Never loosen verse validation to pass a test** — fix data, seeds, or prompts instead.

---

## USEFUL LATER

- Admin backfill loop: `lib/ai/connection-batch.ts`, `connection_coverage.exhaustedAt`, Gemini key rotation (`AGENTS.md`).
- Prompt admin overrides: `lib/ai/prompt-registry.ts` — can change wording, not Tanzih append.
- Provider selection: `lib/ai/ai.ts` → `resolveProvider("connections")`.
- Integration test setup with Testcontainers for full miss-path verification.

---

## IGNORE FOR NOW

- Exact embedding model dimensions and Gemini quota classification (`lib/ai/gemini-errors.ts`) — Phase 9 territory.
- Admin connection review UI workflow — unless you contribute to moderation tooling.
- `connection_coverage` denormalized counts — backfill bookkeeping only.

---

## Phase 7 checkpoint questions

Answer from memory, then verify in code:

1. What function enforces that the model cannot introduce a ref outside the discovery set?
2. Under what two conditions does `generateConnections` (legacy) run?
3. Why does contrast use `semanticCandidates` instead of a separate SQL query?
4. What happens if the model returns well-formed `[]` vs prose refusal?
5. Where is Tanzih injected, and why is it not a `{{placeholder}}`?

---

**Next:** [Phase 8 — State and canvas model (detailed)](./phase-8-state-and-canvas.md) — Zustand stores, persistence, share serialization, and how canvas state relates to (but is not) the Postgres graph.

Say **“continue to Phase 8”** when ready, or ask to zoom into any step above.

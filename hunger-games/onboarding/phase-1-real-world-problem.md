# Phase 1 — What Real-World Problem Does Hunger Games Solve?

> **Open Food Facts Hunger Games — Contributor Onboarding**
>
> This document is Phase 1 of a progressive onboarding series.
> It is based on repository evidence (README, `src/robotoff.ts`, `src/hooks/useQuestions.ts`, `src/const.ts`) and the official [Robotoff API Reference](https://openfoodfacts.github.io/robotoff/references/api/).

---

## Before we open any files

Imagine a supermarket aisle photographed thousands of times by volunteers around the world. Each photo might show a brand logo, a Nutri-Score badge, an ingredients list, a recycling symbol, or a barcode — but the photo is just pixels. Turning those pixels into structured facts ("this product is organic", "brand is Danone", "category is yogurt") is hard at scale.

That is the problem Hunger Games exists to help solve — not by doing the machine learning itself, but by making **human validation of machine guesses** fast enough that millions of products can be improved.

---

## The pipeline (verified model)

Here is the end-to-end story, with each arrow labeled by what we can **prove** versus what we **infer**.

```
Open Food Facts product database
        │
        │  volunteers upload product photos + some manual edits
        ▼
Product images stored on OFF infrastructure
        │
        │  FACT: README states OFF processes pictures with OCR and ML
        ▼
Robotoff generates predictions / insights
        │
        │  FACT: Robotoff API defines "insights" as facts inferred from pictures
        │  FACT: Robotoff exposes /questions/ for items needing human validation
        ▼
Hunger Games fetches a question and shows it as a simple UI
        │
        │  user clicks Yes / No / Skip
        ▼
Hunger Games sends an annotation to Robotoff
        │
        │  FACT: robotoff.annotate(..., update: 1) is called
        ▼
Robotoff records the vote and may update Open Food Facts
        │
        │  FACT (Robotoff API): update=1 sends the update to Open Food Facts
        │  INFERENCE: exact Product Opener field changes depend on insight_type
        ▼
Open Food Facts product record becomes more complete / accurate
```

### Corrections to a naive mental model

| Naive assumption | Verified reality |
|---|---|
| "Hunger Games runs the ML models" | **FACT:** Hunger Games is a React frontend. ML lives in Robotoff. |
| "A question and an insight are the same object" | **FACT:** A question is a **presentation** of an insight that still needs validation. Questions reference `insight_id`. |
| "Clicking Yes immediately writes to Open Food Facts from the browser" | **FACT:** The browser calls **Robotoff's annotate endpoint**. Robotoff decides whether/when to push to OFF (`update=1`). |
| "Every click changes production data instantly" | **FACT (Robotoff API):** Anonymous users contribute **votes**; several identical anonymous votes may be required. Registered OFF users' votes can be applied directly. |
| "Skip means nothing happened" | **FACT (Robotoff API):** `-1` (skip) tells Robotoff not to show that insight again **to that user** (authenticated or not). |

---

## What is Open Food Facts?

**FACT (README + ecosystem context):** Open Food Facts is a collaborative, open database of food (and related) products. Anyone can scan a barcode, photograph a product, and contribute structured data — ingredients, nutrition, labels, packaging, brands, and more.

**INFERENCE:** The project's scale (millions of products, global contributors, multiple languages) means manual full-record editing alone cannot keep up with incoming photos and partial data.

**Why crowdsourced product data matters:** Product labels differ by country, language, and reformulation. A central proprietary database cannot cover every local brand and variant. OFF's model — open data, volunteer contributions, barcode as universal key — makes it possible to aggregate global coverage from distributed effort.

**Why photos alone are insufficient:** A photo proves *something was visible on packaging at capture time*, but not:

- whether the ML interpretation is correct
- whether the value belongs in a taxonomy-normalized field (`en:organic`, not just the word "Bio")
- whether conflicting signals exist across multiple photos of the same product
- whether a detected logo actually refers to the claimed brand vs. a certification mark

Photos are evidence. Structured fields are claims. Robotoff proposes claims; humans validate them.

---

## What does Robotoff do?

**FACT (Robotoff API intro):** "Robotoff provides a simple API allowing consumers to fetch predictions and annotate them."

Robotoff sits between raw OFF assets (especially images) and actionable product facts. Concretely it:

1. **Runs predictors** — ML models, OCR pipelines, logo detectors, etc. (**FACT:** API has a `predictor` filter; README mentions OCR/ML.)
2. **Materializes insights** — **FACT (Robotoff API):** "An insight is a fact about a product that has been either extracted or inferred from the product pictures, characteristics,… If the insight is correct, the Openfoodfacts DB can be updated accordingly."
3. **Exposes questions** — human-friendly prompts derived from insights that still need validation.
4. **Accepts annotations** — Yes / No / Skip / (sometimes) structured data.
5. **Orchestrates OFF updates** — when annotations confirm an insight and policy allows it.

**INFERENCE:** Robotoff is the **system of record for ML-derived candidate facts and their validation state**. Open Food Facts remains the **system of record for published product data**.

---

## Domain vocabulary (Phase 1 level)

These terms will be expanded in Phase 3. For now, keep this distinction:

| Term | One-line meaning | Created by | Hunger Games role |
|---|---|---|---|
| **Prediction** | Raw ML output from a model | Robotoff predictors | Rarely shown directly; filtered into insights |
| **Insight** | A candidate fact about a product (`insight_id`) | Robotoff | Annotated via API; not usually rendered as raw JSON |
| **Question** | UI-ready prompt tied to an `insight_id` | Robotoff (`/questions/`) | Fetched, displayed, answered |
| **Annotation** | Human decision: `1` yes, `0` no, `-1` skip, `2` yes+data | Human via Hunger Games → Robotoff | Sent by `robotoff.annotate()` |
| **Product update** | Change to OFF product record | Product Opener (via Robotoff when allowed) | **Not** direct from Hunger Games except through Robotoff |

**Do not interchange "prediction", "insight", and "question"** — the API treats them as related but distinct layers.

Example question payload shape (**FACT** from `src/robotoff.ts`):

```typescript
interface QuestionInterface {
  barcode: string;
  insight_id: string;
  insight_type: string;   // e.g. "label", "brand", "category"
  question: string;       // e.g. "Does the product have this label?"
  source_image_url?: string;
  ref_image_url?: string; // reference logo/badge when applicable
  type: string;           // e.g. "add-binary"
  value: string;          // human-readable value
  value_tag: string;      // taxonomy tag sent to Product Opener if accepted
}
```

---

## Why does Robotoff need humans at all?

Machine confidence is not the same as ground truth.

**INFERENCE (strong, industry-standard, aligned with Robotoff design):** Models misread blurry photos, confuse similar logos, hallucinate text from OCR noise, and cannot know context (promotional sticker vs. permanent label).

**FACT (Robotoff API):** The annotate endpoint implements a **voting mechanism**. Anonymous annotations are votes; registered users can apply directly. This explicitly encodes distrust of single-shot automated or anonymous decisions for production data.

Humans provide:

- **Precision** — reject false positives before they pollute OFF
- **Recall routing** — confirm true positives so Robotoff can apply updates
- **Ambiguity resolution** — Skip removes bad questions from *your* queue without falsely rejecting
- **Training signal** — validated annotations improve future predictors (**INFERENCE**, standard ML loop; exact retraining pipeline is **UNKNOWN** from Hunger Games repo alone)

---

## Why does Robotoff need Hunger Games specifically?

Robotoff exposes HTTP APIs. It does not ship a consumer-grade annotation UX at the scale OFF needs.

**FACT (README):**

> "Hunger Games is a series of mini-apps that let users contribute data to Open Food Facts, in a rather fun way using React. It relies heavily on Robotoff APIs, as well as Open Food Facts APIs."

> "We process all pictures using top OCR and Machine earning techniques and get a lot of predictions about the products. We then need to let users leverage those predictions to easily complete products, in a fun way, on their desktop and/or mobile devices."

So:

> **Robotoff needs humans.**
> **Humans need an interface optimized for speed.**
> **Hunger Games is that interface.**

Without Hunger Games (or equivalent clients), insights would pile up in Robotoff with no scalable path to validation.

**FACT:** Production URL is `https://hunger.openfoodfacts.org`. Robotoff API base in code: `https://robotoff.openfoodfacts.org/api/v1`.

Hunger Games is **not** Robotoff and **not** the OFF backend. It is a **frontend participant** in a larger ecosystem. Phase 2 will map the boundaries precisely.

---

## Why "game-like" interaction instead of full product editing?

**FACT (README goal):** "Every Open Food Facts user can annotate products in a few minutes."

Full product editing (navigate OFF, find field, type taxonomy value, save) has high cognitive load. Hunger Games reduces each contribution to:

1. Read one sentence question
2. Glance at one cropped image
3. Tap Yes / No / Skip

**INFERENCE:** Game framing (multiple mini-apps, keyboard shortcuts, playful UI per README/Figma links) increases throughput and repeat participation — critical for clearing insight backlogs.

**FACT (README outstanding issues):** Maintainers explicitly worry about "cognitive load on the users for the questions" — confirming UX efficiency is a first-class product concern, not decoration.

---

## What happens when you click Yes, No, or Skip?

This is the most important runtime story for Phase 1. We'll trace the **Questions** mini-game — the architectural core (deeper walkthrough in Phase 7).

### The three buttons map to three integers

**FACT** (`src/const.ts`):

```typescript
export const CORRECT_INSIGHT = 1;   // Yes
export const WRONG_INSIGHT = 0;     // No
export const SKIPPED_INSIGHT = -1;  // Skip
```

**FACT** (`src/pages/questions/QuestionDisplay.tsx`): buttons call `answerQuestion({ question, answer })`.

### Step-by-step: what the frontend does

**FACT** (`src/hooks/useQuestions.ts`, `answerQuestion`):

1. **Fire annotation to Robotoff (non-blocking)**

   ```typescript
   robotoff.annotate(question.insight_id, answer).catch((err) => {
     console.error("Error while answering question", err);
   });
   ```

   **FACT** (`src/robotoff.ts`):

   ```typescript
   annotate(insightId: string, annotation: -1 | 0 | 1) {
     return robotoffClient.annotate({
       insight_id: insightId,
       annotation: annotation,
       update: 1,
     });
   }
   ```

   `update: 1` means Robotoff should send a confirmed update to Open Food Facts when policy allows (**FACT**, Robotoff API).

2. **Immediately update local React Query cache (optimistic UI)**

   The answered question is **removed from the local queue** before the network call completes:

   ```typescript
   queryClient.setQueryData(keys, (data) => ({
     questions: data.questions.filter(
       (q) => q.insight_id !== question.insight_id,
     ),
     count: data.count !== 100 ? data.count - 1 : 100,
   }));
   ```

   Classification: **optimistic for queue progression**, **fire-and-forget for remote mutation** (errors only hit `console.error`).

3. **Maybe prefetch more questions**

   If the local queue drops to ≤5 but the server reports more available, a background mutation refetches and appends non-duplicate questions.

4. **Update recent-answers memory (Yes/No only)**

   Skip is intentionally excluded from the recent-answers sidebar history.

5. **Analytics**

   Matomo event tracked via `useMatomoTrackAnswerQuestion`.

6. **UI shows the next question**

   `question = questions[0]` — the first item in the cached array after removal.

### Step-by-step: what Robotoff does (remote)

**FACT (Robotoff API — Submit an annotation):**

| Click | `annotation` value | Meaning | Effect on OFF data |
|---|---|---|---|
| **Yes** | `1` | Insight is correct | With `update=1`, Robotoff sends update to Open Food Facts when allowed |
| **No** | `0` | Insight is incorrect | Will **not** be applied; negative signal recorded |
| **Skip** | `-1` | User cannot/won't decide | Insight hidden from this user in future; not a rejection of the fact itself |

Additional rules (**FACT**, Robotoff API):

- **Anonymous user:** annotation counted as a **vote**; multiple identical anonymous votes may be required before application.
- **Registered OFF user (session cookie / Basic Auth):** vote can be **applied directly**.
- **Skip tracking:** voting mechanism remembers skipped insights per user/device so the same question does not keep reappearing.

**INFERENCE:** When logged into Open Food Facts in the same browser (credentials included via `fetch(..., { credentials: "include" })` in `robotoff.ts`), your Yes/No may have immediate downstream effect. When anonymous, you are contributing to a consensus.

### What Hunger Games does *not* do on Yes/No/Skip

- Does **not** call OFF Product Opener edit APIs directly for standard binary questions
- Does **not** wait for annotate success before advancing the UI
- Does **not** roll back the UI if annotate fails (only logs error — investigate in Phase 7)

---

## Comparison for your background

If you've built Firebase apps with optimistic UI:

| Pattern you know | Hunger Games equivalent |
|---|---|
| Firestore `onSnapshot` live data | TanStack Query fetching Robotoff `/questions/` |
| Optimistic `setDoc` then reconcile | `setQueryData` removes question immediately; annotate is async |
| Firebase Auth UID on writes | OFF session cookie / anonymous device voting on Robotoff |
| Client writes product truth | **No** — Robotoff owns validation → OFF update path |

The important difference: **server-state owner for annotations is Robotoff, not Hunger Games**. The frontend cache is a **working queue**, not authoritative annotation state.

---

## One compact mental model (keep this in your head)

> **Open Food Facts holds products and photos.**
> **Robotoff turns photos into insights and questions.**
> **Hunger Games turns questions into annotations.**
> **Robotoff turns confirmed annotations into product updates.**

Or shorter:

> **Photos → predictions → questions → annotations → product facts**

You are the human step in the middle. Hunger Games is the arcade cabinet; Robotoff is the referee and scorekeeper; Open Food Facts is the league record book.

---

## Documentation vs code reality (Phase 1 notes)

| Topic | Documented | Code reality | Classification |
|---|---|---|---|
| Package manager | README: "Install yarn" (classic link) | `package.json`: `"packageManager": "yarn@4.18.0"` | **POSSIBLY STALE DOCUMENTATION** — use Yarn 4 / Corepack in practice |
| Default branch in CONTRIBUTING | `upstream/master` | Verify on clone in Phase 17 | **DOCUMENTED WORKFLOW** — confirm still current |
| CORS workaround | README suggests browser extension | Not verified yet | **DOCUMENTED WORKFLOW** — investigate safely in Phase 12 |
| ML typo in README | "Machine earning" | — | Harmless staleness |

---

## Headspace protection

### MUST UNDERSTAND NOW

1. OFF = open product database; photos are evidence, not structured data.
2. Robotoff = ML + insight/question/annotation API; owns validation orchestration.
3. Hunger Games = React frontend; fetches questions, sends annotations.
4. Insight ≠ question ≠ annotation — related layers, different identifiers/API objects.
5. Yes=`1`, No=`0`, Skip=`-1`; annotate goes to Robotoff with `update: 1`.
6. UI advances optimistically; remote failure currently only logs to console.

### USEFUL LATER

- Exact insight types (`label`, `brand`, `category`, …) and per-type OFF field mapping
- Logo annotation flows (separate API methods in `robotoff.ts`)
- Voting thresholds for anonymous users
- TanStack Query refill logic edge cases
- Authentication/session setup for local dev

### IGNORE FOR NOW

- Individual mini-games beyond the core Questions flow (logos, nutrition, packaging, …)
- Taxonomy file generation scripts (`yarn countries`, `yarn nutriments`)
- CI/CD, Netlify, Crowdin workflows
- TypeScript migration boundaries across `.jsx` / `.tsx`

---

## Phase 1 summary — remember these five things

1. **The problem:** scale — millions of product photos need human validation of ML-extracted facts.
2. **The division of labor:** Robotoff thinks; humans confirm; OFF stores published truth.
3. **Hunger Games' job:** minimize friction from insight to annotation through game-like UX.
4. **The click path:** UI optimistic removal → `robotoff.annotate(insight_id, ±1|0, update=1)` → Robotoff may update OFF.
5. **The boundary:** Hunger Games never replaces Robotoff or OFF; it consumes their APIs.

### Uncertainties to resolve in later phases

- **UNKNOWN:** Exact anonymous vote threshold before OFF update (Robotoff server config).
- **UNKNOWN:** Full mapping of each `insight_type` to Product Opener fields (requires Robotoff source or docs).
- **UNKNOWN:** Whether local dev works without CORS workarounds for all endpoints (Phase 12).
- **INFERENCE:** How often insights become questions vs. auto-applied — needs Robotoff-side investigation.

---

## What comes next

**Phase 2 — The Open Food Facts ecosystem**

We will place Hunger Games in a system-context diagram alongside:

- Robotoff
- Open Food Facts APIs / product database
- `@openfoodfacts/openfoodfacts-nodejs`
- `openfoodfacts-webcomponents`
- static taxonomy services

For each dependency: who owns source of truth, who calls whom, and what crosses the boundary.

---

## Primary sources consulted

| Source | What we verified |
|---|---|
| `README.md` | Project purpose, ecosystem links, dev setup claims |
| `src/robotoff.ts` | Question shape, annotate call, API URL, credentials |
| `src/hooks/useQuestions.ts` | Optimistic queue, annotate fire-and-forget, refill |
| `src/const.ts` | Annotation integer constants, API URLs |
| `src/pages/questions/QuestionDisplay.tsx` | Yes/No/Skip wiring |
| [Robotoff API Reference](https://openfoodfacts.github.io/robotoff/references/api/) | Insight definition, annotate semantics, voting, `update` flag |

---

*Stop here. Read this once, question anything that feels wrong, then we continue to Phase 2.*

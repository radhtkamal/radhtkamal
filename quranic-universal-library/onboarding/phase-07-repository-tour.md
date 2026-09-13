# Phase 7 — Repository Tour

> **Onboarding series:** progressive deep-dive for becoming a legitimate QUL contributor.  
> **Prerequisites:** [Phases 1–6](phase-01-what-is-qul.md)  
> **This file:** where things live in the repo, mapped to the architecture layers from Phase 6.

---

## How to use this tour

This is **not** a file tree dump. It is a **navigational map**: for each major folder, you get:

- **What it is** (role in the system)
- **When you touch it** (as a contributor)
- **What to read first** (entry points)

Counts below are approximate snapshots from the repo at onboarding time (~1,200+ files under `app/`, ~160 under `lib/`).

---

## Top-level mental map

```text
quranic-universal-library/
├── app/           ← Rails application (HTTP, models, views, jobs, admin)
├── lib/           ← Bulk domain logic (import/export/audio/corpus) + rake tasks
├── config/        ← Routes, DB, Sidekiq, docs manifest, initializers
├── db/            ← CMS migrations + schema.rb ONLY (not Quran tables)
├── bin/           ← setup, dev, rails wrappers
├── docker/        ← Dev DB init scripts
├── test/          ← Thin test suite (mostly lib/services)
├── scripts/       ← One-off ops scripts (fonts, audio, screenshots)
├── public/        ← Static assets served directly
├── onboarding/    ← This series (local, not yet in upstream docs)
├── docs/          ← Almost empty; NOT the docs source of truth
└── .github/       ← CI (CodeQL, Docker deploy to K8s)
```

**STALE DOC FLAG:** `app/views/docs/markdown/contributing.md` says *"Edit files in `docs/` first"* — **FACT:** canonical docs live in `app/views/docs/markdown/` and are wired through `DocsManifest` + `config/docs.yml`.

---

## `app/` — the Rails surface (~1,222 files)

Everything user-facing or request-handling lives here. Think of `app/` as **four concentric rings**:

```text
         ┌─────────────────────────────────────┐
         │  views/ + javascript/ + assets/     │  ← what users see
         ├─────────────────────────────────────┤
         │  controllers/ + presenters/ +       │  ← HTTP + shaping
         │  finders/ + helpers/ + components/  │
         ├─────────────────────────────────────┤
         │  models/ + services/                │  ← data + small services
         ├─────────────────────────────────────┤
         │  admin/ + jobs/ + mailers/          │  ← CMS + async + email
         └─────────────────────────────────────┘
```

### `app/controllers/` (~52 files) — HTTP entry points

| Area | Path | Routes to know |
|---|---|---|
| **Public catalog** | `resources_controller.rb`, `landing_controller.rb`, `search_controller.rb` | `/`, `/resources`, `/search` |
| **Docs** | `community_controller.rb` | `/docs`, `/docs/:key`, `/tools` |
| **Contributor tools** | `translation_proofreadings_controller.rb`, `tafsir_proofreadings_controller.rb`, `mushaf_layouts_controller.rb`, `morphology_phrases_controller.rb`, `word_text_proofreadings_controller.rb`, … | See `config/routes.rb` |
| **Audio tooling** | `surah_audio_files_controller.rb`, `ayah_audio_files_controller.rb` | Segment timestamp editors |
| **Segments (Vue)** | `segments/` namespace | `/segments/*`, `/segment_pipeline/*` |
| **API v1** | `api/v1/` | `/api/v1/chapters`, translations, tafsir, audio |
| **Morphology public** | `morphology/` | `/morphology/roots/:id`, lemmas, treebank |
| **Auth** | `users/` (Devise overrides) | `/users/sign_in`, registrations |
| **Health** | `health_controller.rb` | `/health` |

**Contributor pattern:** find the URL in `config/routes.rb` → open the controller → follow to presenter/model/service.

**FACT** — `CommunityController#tools` builds the contributor tool grid from `ToolsHelper#developer_tools` (hard-coded `ToolCard` list, not DB-driven).

### `app/admin/` (~113 files) — Active Admin CMS

Organized by domain, mirroring editorial structure:

| Subfolder | What admins manage |
|---|---|
| `content/` | `ResourceContent`, change logs, chapter info, root details |
| `quran/` | Verses, words, mushaf pages/words, tajweed, tokens |
| `audio/` | Recitations, chapter audio files, metadata jobs |
| `draft/` | Draft translations, tafsirs, word translations |
| `downloads/` | `DownloadableResource`, export refresh buttons |
| `morphology/` | Grammar, graph nodes, phrase data |
| `settings/` | Languages, API clients, resource tags, char types |
| `tools/` | Data integrity checks, diagnostics |
| `dictionary/`, `grammar/`, `topic.rb`, … | Supporting reference data |

**When you touch it:** debugging editorial workflows, adding admin fields/filters, wiring new export buttons.

**Entry point for Phase 8:** `app/admin/content/resource_content.rb` — the hub for approve/import/export actions.

### `app/models/` (~147 files) — Active Record

Two base classes split the world (Phase 5):

| Base | Examples |
|---|---|
| `ApplicationRecord` | `User`, `DownloadableResource`, `Draft::Translation`, `UserProject`, `ResourcePermission` |
| `QuranApiRecord` | `Verse`, `Word`, `Translation`, `Tafsir`, `ResourceContent`, `Mushaf`, `Recitation`, `Morphology::Word` |

**Key concerns** (`app/models/concerns/`):

| Concern | Purpose |
|---|---|
| `Resourceable` | Scopes content rows to `resource_content_id` |
| `HasMetaData` | JSON `meta_data` column helpers on `ResourceContent` |
| `PaperTrailAttribution` | Who changed what |
| `Slugable` | URL slugs for downloadable resources |
| `NavigationSearchable` | Admin/global search indexing |

**Namespaces to know:**

- `app/models/draft/` — CMS-side draft rows before publish
- `app/models/morphology/` — Linguistic graph (phrases, grammar, dependency)
- `app/models/audio/` — Audio recitation sub-models
- `app/models/segments/` — Segment tooling models + dynamic SQLite connection
- `app/models/tools/` — Integrity check classes (not HTTP controllers)

### `app/presenters/` (~44 files) — View models

**FACT** — QUL uses presenters heavily. Controllers set `@presenter` or call presenter classes directly.

| Pattern | Example |
|---|---|
| Page presenters | `ResourcesPresenter`, `LandingPresenter` |
| Resource shaping | `TranslationPresenter`, `TafsirPresenter`, `MushafLayoutResourcesPresenter` |
| API v1 | `app/presenters/v1/chapter_presenter.rb`, `verse_presenter.rb` |

**INFERENCE:** When a view needs computed keys, grouped data, or CDN URLs, look for a presenter before reading the ERB.

### `app/services/` (~26 files) — Small orchestration

Not the main domain layer — that's `lib/`. Services here are **app-scoped glue**:

| Service | Role |
|---|---|
| `DocsPageService` / `DocsManifest` | Load markdown docs from `app/views/docs/markdown/` |
| `Resources::SearchQuery` | Resource catalog search |
| `Morphology::PhraseNodeService` | Phrase graph mutations |
| `Search::ArabicNormalizer` | Arabic text normalization for search |

**Rule of thumb:** if it's bulk import/export or file generation → `lib/`. If it's request-scoped orchestration → `app/services/`.

### `app/jobs/` (~30 files) — Sidekiq workers

Grouped by concern:

| Folder / file | Role |
|---|---|
| `draft_content/` | Approve/import/check draft content |
| `export/` | Tafsir, mushaf layout exports |
| `audio/` | Generate files, split gapless, segment export |
| `segments/` | Reciter segment export |
| `recurring/` | API stats updates |
| `async_resource_action_job.rb` | Generic `resource.send(action)` dispatcher |
| `backup_job.rb` | Daily DB backup (prod only) |

**Trace pattern:** admin button → `perform_later` → job → `lib/` class → model/S3 update.

### `app/views/` (~509 files) — ERB templates

High-signal folders:

| Folder | Contents |
|---|---|
| `landing/` | Homepage |
| `resources/` | Catalog, previews per resource type (`previews/_*.html.erb`) |
| `docs/` | Docs chrome + `markdown/*.md` **source files** |
| `community/` | Tools index, SVG optimizer shell |
| `translation_proofreadings/`, `tafsir_proofreadings/` | Contributor proofreading UI |
| `morphology_phrases/`, `mushaf_layouts/` | Community editing tools |
| `segments/` | Segment dashboard (hosts Vue mount points) |
| `api/v1/` | Jbuilder/JSON props for API responses |
| `tools/header_alert/` | Per-tool contextual help banners |
| `shared/` | Cross-cutting partials (ayah display, search, mushaf SVG) |

**FACT** — Resource preview partials under `app/views/resources/previews/` map 1:1 to `DownloadableResource::RESOURCE_TYPES`.

### `app/javascript/` (~148 files) — Frontend

| Path | Tech | When |
|---|---|---|
| `controllers/` (~91 Stimulus controllers) | Stimulus | Most interactive contributor tools |
| `segments/` | Vue 3 + esbuild | Audio segment builder |
| `svg/` | Vue 3 | SVG optimizer tool |
| `active_admin/` | jQuery helpers | CMS-only JS |
| `application.js` | Entry | Turbo + Stimulus registration |

**Build:** `package.json` → esbuild (`npm run build`) + Tailwind (`npm run build:css`). Dev reload via `Procfile.dev`.

### Smaller `app/` folders

| Folder | Role |
|---|---|
| `app/finders/v1/` | API query objects (e.g. `SegmentFinder`) |
| `app/components/` | ViewComponent (minimal — `SplitScreenComponent`) |
| `app/helpers/` | View helpers; `tools_helper.rb` registers contributor tools |
| `app/mailers/` | Export update emails, segment completion, user mail |
| `app/assets/stylesheets/` | SCSS/Tailwind source, Active Admin overrides |
| `app/channels/` | Action Cable (if used — low visibility in onboarding) |

---

## `lib/` — domain pipelines (~164 files)

**This is where senior contributors spend time** for data work.

```text
lib/
├── importer/          ← Pull external sources INTO Quran DB
├── exporter/          ← Serialize Quran DB TO files
├── audio/             ← Audio file generation, gapless split, metadata
├── audio_segment/     ← Segment format adapters (tarteel, ayah-by-ayah, …)
├── layout_exporter/   ← Mushaf page layout math
├── corpus/            ← Morphology type enums / grammar HTML
├── tajweed_annotation/← Tajweed tokenization
├── utils/             ← Shared: sanitizers, search, backups, Quran helpers
├── tasks/             ← Rake tasks (28 files) — one-off migrations & imports
├── api/               ← API param helpers
├── active_admin/      ← Custom Active Admin inputs
├── data/              ← Static JSON seed data
└── (top-level)        ← export_service.rb, tool_card.rb, cloudflare_cache_clearer.rb, …
```

### `lib/importer/` (9 classes)

| Class | Source |
|---|---|
| `Importer::QuranEnc` | QuranEnc translations |
| `Importer::QuranEncTafsir` | QuranEnc tafsir |
| `Importer::IslamEnc` | IslamEnc |
| `Importer::QuranAcademy`, `QuranKsuEduTafsir`, `QuranTafsirNet`, `TafsirApp`, `EQuranLibrary` | Various tafsir/translation sources |
| `Importer::Base` | Shared import scaffolding |

**Entry point:** `Importer::Base` + one concrete importer when tracing a specific source.

### `lib/exporter/` (26 classes)

| Class | Exports |
|---|---|
| `Exporter::DownloadableResources` | **Orchestrator** — `export_all`, `refresh_export!` dispatch |
| `Exporter::ExportTranslation` | Translation JSON/SQLite variants |
| `Exporter::ExportTafsir` | Tafsir packages |
| `Exporter::ExportMushafLayout` | Mushaf layout data |
| `Exporter::ExportSurahRecitation` / `ExportAyahRecitation` / `ExportWordRecitation` | Audio manifests |
| `Exporter::ExportQuranicMorphology` | Morphology dumps |
| `Exporter::ExportFont` | Font files |
| `Exporter::BaseExporter` | Shared export utilities |

**FACT** — `Exporter::DownloadableResources#create_download_file` zips output and attaches to `DownloadableFile` via Active Storage.

### `lib/tasks/` — operational rake tasks

Representative tasks (not exhaustive):

| Task file | Purpose |
|---|---|
| `dump.rake` | Database dumps |
| `import_*.rake` | One-off source imports (qaloun, warsh, lemmas, topics) |
| `audio.rake`, `audio_segments.rake` | Audio pipeline maintenance |
| `quran_scripts.rake` | Script variant fixes |
| `migrate_*.rake` | Data migrations outside AR migrations |
| `tailwindcss.rake` | CSS build hook |

**INFERENCE:** Many tasks are **run manually in production/staging** by maintainers, not CI.

---

## `config/` — wiring (~64 files)

| File / folder | What it controls |
|---|---|
| `routes.rb` | **Start here** for any URL hunt |
| `database.yml` | Two-DB connections (CMS + Quran) |
| `storage.yml` | Active Storage services (`local`, `qul_exports`, `qul_segments`, `database_backups`) |
| `docs.yml` | Docs sidebar manifest (categories + page slugs) |
| `initializers/sidekiq.rb` | Job adapter, scheduler, Sidekiq Web |
| `sidekiq_scheduler.yml` | Cron: daily backup, weekly QuranEnc check |
| `resources/` | `surah_name_aliases.yml` and resource config |
| `environments/` | Memcached in prod, null cache in dev test |
| `locales/` | i18n (limited surface) |

---

## `db/` — CMS schema only

| Path | Role |
|---|---|
| `schema.rb` | **CMS tables only** (~40 tables) — users, downloads, drafts, versions |
| `migrate/` (~105 files) | CMS migrations |
| `seeds.rb` | Minimal seed data |

**FACT** — No `schema.rb` for Quran tables. Quran schema comes from SQL dump (Phase 17).

---

## `bin/` + `docker/` + root process files

| File | Role |
|---|---|
| `bin/setup` | `bundle install`, `db:create:all`, `db:prepare`, `npm install` |
| `bin/dev` | Foreman → Rails + esbuild watch + Tailwind watch |
| `Procfile.dev` | Process definitions for dev |
| `docker/dev/init-db.sh` | Creates `quran_dev` DB + `quran` schema |
| `Dockerfile` | Production image (used by GitHub Actions deploy) |

---

## Documentation — three locations, one truth

| Location | Status |
|---|---|
| `app/views/docs/markdown/*.md` | **FACT — canonical source** rendered at `/docs/:key` |
| `config/docs.yml` | **FACT — navigation manifest** (categories, ordering) |
| `docs/` (repo root) | **FACT — nearly empty** (`superpowers/` only); ignore for user docs |
| `README.md` | Links to live site docs at qul.tarteel.ai/docs |
| `.github/copilot-instructions.md` | Maintainer/agent setup guide (detailed env instructions) |

**Services involved:** `DocsManifest` reads `config/docs.yml`; `DocsPageService` loads markdown files.

---

## `test/` — thin coverage (~18 files)

| Path | What's tested |
|---|---|
| `test/lib/` | Some corpus/morphology lib code |
| `test/services/` | A few service objects |
| `test/test_helper.rb` | Standard Rails test setup |

**INFERENCE:** Quran models are largely **stubbed or untested** in integration tests. Manual verification and admin diagnostics (`Tools::DataIntegrityChecks`) compensate.

**UNKNOWN:** Full test strategy — Phase 20 will dig deeper.

---

## `scripts/` — ops utilities (~22 files)

Not part of runtime. Used for font subsetting, audio conversion, screenshots, benchmarks:

- `genrate_font_glyph.rb`, `subset_fonts.sh`, `woff2.rb` — font pipeline
- `mp3_to_wave.sh`, `optimize_audio.sh` — audio prep
- `cypress-e2e/` — browser test scaffolding
- `screenshots.py` — docs screenshots

**Contributor rule:** don't call these from app code; they're manual/CI helpers.

---

## Navigation cheat sheet — "I need to find X"

| I need to… | Start here |
|---|---|
| Add a public URL | `config/routes.rb` → new controller action → view |
| Change resource download page | `app/controllers/resources_controller.rb` + `app/views/resources/` |
| Fix stale JSON on `/resources` | `DownloadableResource#refresh_export!` → `lib/exporter/downloadable_resources.rb` |
| Trace an import | `app/admin/content/resource_content.rb` → `lib/importer/` |
| Add admin field/filter | `app/admin/<domain>/` |
| Understand permissions | `app/models/ability.rb` + `ResourcePermission` |
| Find contributor tool list | `app/helpers/tools_helper.rb` |
| Edit website docs | `app/views/docs/markdown/` + `config/docs.yml` |
| Add Stimulus behavior | `app/javascript/controllers/` |
| Add Vue UI | `app/javascript/segments/` or `svg/` |
| Debug background work | `app/jobs/` + Sidekiq Web (`/sidekiq`) |
| Run one-off data fix | `lib/tasks/*.rake` |
| See CMS table shape | `db/schema.rb` |
| See Quran table shape | Load dump locally (Phase 17); grep models |

---

## Three walkthrough recipes (practice now)

### Recipe A — "Where does translation proofreading live?"

```text
/tools  →  ToolsHelper#developer_tools (ToolCard entry)
       →  /translation_proofreadings  (routes.rb)
       →  TranslationProofreadingsController
       →  Draft::Translation (CMS model)
       →  admin approve  →  DraftContent::ApproveDraftTranslationJob
       →  Translation (Quran DB)
       →  refresh_export!  →  lib/exporter/export_translation.rb
```

### Recipe B — "Where is mushaf layout export?"

```text
/mushaf_layouts  (contributor tool)
/cms  →  admin/quran/mushaf*.rb
      →  Export::MushafLayoutExportJob
      →  lib/exporter/export_mushaf_layout.rb
      →  lib/layout_exporter/
      →  DownloadableFile (S3)
```

### Recipe C — "Where is API chapter list?"

```text
GET /api/v1/chapters
  →  app/controllers/api/v1/chapters_controller.rb
  →  app/presenters/v1/chapter_presenter.rb
  →  app/views/api/v1/chapters/*.json.props
  →  Chapter < QuranApiRecord
```

---

## Repo scale signals (what the numbers mean)

| Signal | Implication for you |
|---|---|
| 113 admin files | CMS is mature; learn Active Admin patterns early |
| 91 Stimulus controllers | Contributor tools are JS-heavy but not React-first |
| 26 exporters vs 9 importers | Export surface > import surface; exports are the public contract |
| 105 CMS migrations | Schema evolves; Quran data does not migrate via Rails |
| 2 Vue entry points only | Don't expect a component library — it's ERB-first |

---

## Uncertainties

| # | Question | Label |
|---|---|---|
| 1 | Full list of Quran DB tables | **UNKNOWN** until dump loaded (Phase 17) |
| 2 | Whether `docs/superpowers/` is used in production | **UNKNOWN** — only 1 file at repo root `docs/` |
| 3 | Complete Cypress e2e coverage | **UNKNOWN** — scripts exist, scope unclear |
| 4 | Which rake tasks are still run in prod vs historical | **INFERENCE** — many `one_time`/`migrate_*` look legacy |

---

## Phase 7 summary

The repo splits cleanly:

- **`app/`** = HTTP, CMS, views, thin orchestration, jobs
- **`lib/`** = import/export/audio/corpus pipelines (the real domain engine)
- **`config/` + `db/`** = CMS wiring and schema
- **`app/views/docs/markdown/`** = docs source (not `docs/`)

When lost: **`routes.rb` → controller → presenter/model → `lib/` if data moves**.

---

## Stop here — questions before Phase 8

Phase 8 will walk **one representative CMS flow** (`ResourceContent` through Active Admin: draft → approve → export).

Before that:

1. Pick one contributor tool from `/tools` — can you trace it using Recipe A/B/C patterns?
2. Which folder would you check first for a bug in exported SQLite format vs a bug in the proofreading form UI?
3. Any folder from this map that still feels like a black box?

Reply with questions, or say **"proceed"** for **Phase 8 — Active Admin / CMS Flow**.

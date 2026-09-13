# Phase 23 — Contribution Surface Map & Readiness

> **Onboarding series:** progressive deep-dive for becoming a legitimate QUL contributor.  
> **Prerequisites:** [Phases 1–22](phase-01-what-is-qul.md)  
> **This file:** where you can actually help, matched to skills — plus an honest readiness checklist to know when you're prepared to contribute.

---

## You are here

If you've read Phases 1–22, you have a **working mental model** of QUL:

```text
Factory (Rails CMS + tools)  →  curated Quran DB  →  exported zips at /resources
         ↑                              ↑
    two Postgres DBs              verse_key / location joins
    draft-first editing           integrity checks in /cms
```

This final phase turns that knowledge into **actionable entry points** and a **self-assessment**.

---

## Contribution surface map (by skill)

### Profile A — React / TypeScript / Node engineer (you)

Best fits: **frontend tooling, API consumers, service tests, docs, small Rails glue** — not deep ActiveRecord on day one.

| Surface | Entry URL / path | Code area | Risk | Start here? |
|---|---|---|---|---|
| **Docs site** | `/docs` | `app/views/docs/markdown/`, `config/docs.yml` | Low | **Yes** — fix stale `contributing.md` paths |
| **Resource catalog UI** | `/resources` | `ResourcesController`, `app/views/resources/` | Medium | After Phase 18 trace |
| **Ayah viewer** | `/ayah/2:255` | `AyahController`, Turbo frames, Stimulus | Medium | Good second project |
| **Segment builder (Vue)** | `/surah_audio_files/.../segment_builder` | `app/javascript/segments/` | Medium–High | If you know audio UX |
| **Search / parsers** | `/resources?q=…` | `app/services/resources/`, `test/services/` | Medium | **Yes** — has unit tests |
| **Segment validator** | contributor + CMS | `app/services/audio/segment_validator.rb` | Medium | **Yes** — tested service |
| **Stimulus tools** | `/tools` | `app/javascript/controllers/` | Low–Medium | UI bugs, a11y |
| **JSON API v1** | `/api/v1/*` | `app/controllers/api/v1/` | High | Issue first — public contract |
| **Exporters** | CMS export buttons | `lib/exporter/` | High | Maintainer pairing |
| **Import pipelines** | CMS import | `lib/importer/` | Critical | Not solo early |
| **Active Admin** | `/cms` | `app/admin/` | Medium | After CMS flow (Phase 8) |

**INFERENCE:** Your fastest path to a merged PR is **docs + service tests + Stimulus** — areas that match your stack and have thin CI coverage but clear review criteria.

---

### Profile B — Data / content contributor

Best fits: **proofreading, audio segmentation, mushaf, morphology tagging** — minimal Ruby required.

| Surface | Tool URL | Writes to | Needs access |
|---|---|---|---|
| Translation proofreading | `/translation_proofreadings` | `draft_translations` (CMS) | Login + `UserProject` or super_admin |
| Tafsir proofreading | `/tafsir_proofreadings` | draft tafsirs | Same |
| Word translation | `/word_translations` | draft word translations | Same |
| Surah audio segments | `/surah_audio_files` | `audio_segments` (Quran) | Audio annotator or project |
| Ayah audio segments | `/ayah_audio_files` | `audio_files.segments` | Same |
| Mushaf layouts | `/mushaf_layouts` | `mushaf_words` (Quran) | Project access |
| Tajweed annotation | `/tajweed_words` | tajweed tables | Project access |
| Mutashabihat | `/morphology_phrases` | morphology phrases | Project access |
| Dependency graphs | `/morphology/dependency-graphs` | graph nodes/edges | Project access |
| Word concordance | `/word_concordance_labels` | morphology labels | Project access |
| Script proofreading | `/word_text_proofreadings` | script text | Project access |

**Report issues without CMS:** GitHub issue templates (translation, mushaf, script).

**Get access:** Contributor application issue + Discord for large efforts (Phase 21).

---

### Profile C — Rails / backend engineer

| Surface | Path | Notes |
|---|---|---|
| Sidekiq jobs | `app/jobs/` | Approve, export, import — Phase 15 |
| Models / concerns | `app/models/` | Know `ApplicationRecord` vs `QuranApiRecord` |
| Importers | `lib/importer/` | Phase 10 |
| Exporters | `lib/exporter/` | Phase 11 |
| Rake tasks | `lib/tasks/` | One-off data ops |
| CMS actions | `app/admin/` | Wire buttons → jobs |
| Integrity checks | `app/models/tools/data_integrity_checks.rb` | Add SQL diagnostics |

---

### Profile D — DevOps / platform

| Surface | Path | Notes |
|---|---|---|
| Docker / deploy | `Dockerfile`, `.github/workflows/deploy.yml` | Production image |
| CodeQL | `.github/workflows/codeql.yml` | Only CI gate today |
| Sidekiq / Redis | `config/sidekiq.yml`, `docker-compose.yml` | Worker ops |
| S3 storage | `config/storage.yml`, `.env.sample` | Export uploads |
| DB backups | `BackupJob`, `lib/utils/db_backup.rb` | Scheduled |

**Gap opportunity:** Add CI job for `bin/rails test` + `rubocop` — would help all contributors (Phase 20).

---

## Code map → onboarding phases

Quick reference when you're lost:

| If you're working on… | Re-read |
|---|---|
| What QUL is | [Phase 1](phase-01-what-is-qul.md) |
| `verse_key`, `ResourceContent` | [Phase 2](phase-02-quranic-data-model.md) |
| Drafts, PaperTrail | [Phase 3](phase-03-provenance-and-integrity.md) |
| Rails basics | [Phase 4](phase-04-rails-for-react-node-engineers.md) |
| Two databases | [Phase 5](phase-05-two-database-architecture.md) |
| System diagram | [Phase 6](phase-06-architecture-mental-model.md) |
| Folder layout | [Phase 7](phase-07-repository-tour.md) |
| CMS approve flow | [Phase 8](phase-08-cms-flow.md) |
| Contributor edit path | [Phase 9](phase-09-core-runtime-flow.md) |
| Imports | [Phase 10](phase-10-import-pipeline.md) |
| Exports / `/resources` | [Phase 11](phase-11-export-pipeline.md) |
| Morphology | [Phase 12](phase-12-morphology-arabic-linguistic-data.md) |
| Mushaf | [Phase 13](phase-13-mushaf-layouts.md) |
| Audio | [Phase 14](phase-14-audio-subsystem.md) |
| Sidekiq | [Phase 15](phase-15-background-jobs-sidekiq.md) |
| Frontend | [Phase 16](phase-16-frontend-reality.md) |
| Local setup | [Phase 17](phase-17-local-setup.md) |
| URL → code | [Phase 18](phase-18-browser-code-correlation.md) |
| Hands-on experiment | [Phase 19](phase-19-controlled-learning-experiment.md) |
| Tests / integrity | [Phase 20](phase-20-testing-and-data-integrity.md) |
| Culture / PRs | [Phase 21](phase-21-engineering-culture-pr-expectations.md) |
| Git workflow | [Phase 22](phase-22-oss-git-workflow.md) |

---

## Readiness tiers (honest self-assessment)

Check each box. **Tier 3 = ready for meaningful code PRs.** Tier 4 = ready for data pipeline work.

### Tier 1 — Consumer literacy (no local setup)

- [ ] I can explain QUL vs `/resources` downloads
- [ ] I know `verse_key` format (`"2:255"`)
- [ ] I understand translations are packages keyed by `resource_content_id`
- [ ] I won't hotlink `audio-cdn.tarteel.ai` in production apps

**You can:** Use QUL data in your apps; report data issues via GitHub templates.

---

### Tier 2 — Local observer

- [ ] `bin/setup` + mini dump loaded (`Verse.count > 0`)
- [ ] `bin/dev` runs; `/ayah/2:255` loads
- [ ] I can log into `/cms` with seed admin
- [ ] I traced one URL to controller + view (Phase 18)

**You can:** Explore locally; fix docs; open informed issues.

---

### Tier 3 — Code contributor (recommended minimum)

- [ ] Completed Phase 19 experiment OR equivalent draft trace
- [ ] I know CMS DB vs Quran DB table locations
- [ ] I ran `bin/rails test` on a relevant file
- [ ] I can open a focused PR with test steps (Phase 20–22)
- [ ] I know not to edit root `docs/` for website docs

**You can:** Docs PRs, service tests, Stimulus fixes, search/parser improvements.

---

### Tier 4 — Data pipeline contributor

- [ ] Tier 3 complete
- [ ] Sidekiq + Redis running locally
- [ ] Traced import OR export once in logs (Phases 10–11)
- [ ] Ran at least one CMS integrity check
- [ ] Coordinated with maintainers (issue/Discord) for bulk data

**You can:** Approve/import/export work with review; scoped CMS editing.

---

### Tier 5 — Maintainer-shaped (aspirational)

- [ ] Tier 4 complete
- [ ] Comfortable reading `lib/importer/` and `lib/exporter/` end-to-end
- [ ] Understand `segment_locked`, export skip rules, cross-DB anomalies (mushaf line alignment)
- [ ] Can add an integrity check or approve job extension safely

**You can:** Own a resource type's full lifecycle.

---

## Recommended first contributions (by tier)

### Tier 2 → 3 (code)

| PR | Files | Why good |
|---|---|---|
| Fix `contributing.md` docs path | `app/views/docs/markdown/contributing.md` | High value, zero risk |
| Fix "Purpose changes" button | `translation_proofreadings/edit.html.erb` | One line, user-visible |
| Add segment validator test case | `test/services/audio/segment_validator_test.rb` | Matches repo test style |
| Improve `project-setup.md` | `app/views/docs/markdown/project-setup.md` | You hit setup pain → fix it |

### Tier 3 → 4 (code + data awareness)

| PR | Area |
|---|---|
| Improve resource search edge case | `app/services/resources/search_query.rb` |
| Stimulus UX fix on `/resources` | `app/javascript/controllers/` |
| Add CI workflow for `rails test` | `.github/workflows/` (discuss in issue first) |
| New integrity check (SQL) | `app/models/tools/data_integrity_checks.rb` |

### Tier 2 → 3 (data, no Ruby)

| Action | Channel |
|---|---|
| Translation typo report | GitHub translation issue template |
| Proofread one ayah (draft) | `/translation_proofreadings` after access |
| Validate audio segments | `/surah_audio_files` segment builder |

---

## What you should still expect to learn on the job

No onboarding series replaces these — you'll pick them up through PRs:

| Topic | Why it stays fuzzy |
|---|---|
| Full Quran schema | Only in loaded dump; not in `schema.rb` |
| Per-resource approve quirks | Tafsir groups, footnotes, word translations differ |
| Production S3 / CDN config | `.env` + maintainer knowledge |
| `segment_pipeline` routes | Controller missing in repo |
| Which `resource_content_id` values exist in mini dump | Dump-specific subset |
| Maintainer priority queue | Volunteer bandwidth |

**That's normal.** Legitimate contributors ask questions in issues/PRs — with context from this series.

---

## Anti-patterns (don't call yourself ready if…)

| Signal | Reality |
|---|---|
| "I'll fix the translation directly in SQL" | Bypasses draft → approve → export |
| "I'll add a Vue app for every tool" | Wrong frontend pattern (Phase 16) |
| "CI passed so we're good" | CI doesn't run tests (Phase 20) |
| "I'll import a new translation via rake without issue" | Licensing + provenance risk |
| "I haven't loaded the dump but I'll edit exporters" | Can't verify output |

---

## Full phase index

| # | Title | File |
|---|---|---|
| 1 | What is QUL? | `phase-01-what-is-qul.md` |
| 2 | Quranic data model | `phase-02-quranic-data-model.md` |
| 3 | Provenance & integrity | `phase-03-provenance-and-integrity.md` |
| 4 | Rails for React/Node engineers | `phase-04-rails-for-react-node-engineers.md` |
| 5 | Two-database architecture | `phase-05-two-database-architecture.md` |
| 6 | Architecture mental model | `phase-06-architecture-mental-model.md` |
| 7 | Repository tour | `phase-07-repository-tour.md` |
| 8 | CMS flow | `phase-08-cms-flow.md` |
| 9 | Core runtime flow (editing) | `phase-09-core-runtime-flow.md` |
| 10 | Import pipeline | `phase-10-import-pipeline.md` |
| 11 | Export pipeline | `phase-11-export-pipeline.md` |
| 12 | Morphology & Arabic linguistic data | `phase-12-morphology-arabic-linguistic-data.md` |
| 13 | Mushaf layouts | `phase-13-mushaf-layouts.md` |
| 14 | Audio subsystem | `phase-14-audio-subsystem.md` |
| 15 | Background jobs (Sidekiq) | `phase-15-background-jobs-sidekiq.md` |
| 16 | Frontend reality | `phase-16-frontend-reality.md` |
| 17 | Local setup | `phase-17-local-setup.md` |
| 18 | Browser ↔ code correlation | `phase-18-browser-code-correlation.md` |
| 19 | Controlled learning experiment | `phase-19-controlled-learning-experiment.md` |
| 20 | Testing & data integrity | `phase-20-testing-and-data-integrity.md` |
| 21 | Engineering culture & PR expectations | `phase-21-engineering-culture-pr-expectations.md` |
| 22 | OSS git workflow | `phase-22-oss-git-workflow.md` |
| 23 | **Contribution surface & readiness** | `phase-23-contribution-surface-readiness.md` |

---

## One-page cheat sheet (print this)

```text
PRODUCT     Exported files at /resources — not live DB access
DATABASES   quran_community_tarteel (CMS) + quran_dev (Quran)
IDENTITY    verse_key, location, resource_content_id
EDIT PATH   draft (CMS) → approve (job) → refresh_export! (job) → S3
EXCEPTIONS  mushaf + audio segments often write Quran DB directly
DOCS        app/views/docs/markdown/ + config/docs.yml
TESTS       bin/rails test (local); CMS integrity checks for data
PR          fork → upstream/main → small branch → issue-linked PR
FIRST PR    docs fix or service test or typo
```

---

## Final readiness question

Answer without looking:

1. Where do published translations live — CMS or Quran DB?
2. What three commands get a new contributor from clone to browsing an ayah?
3. What's the safest first code PR for a React engineer?

If you answered **Quran DB**, **`bin/setup` → load dump → `bin/dev`**, and **docs or service test** — you're ready to contribute.

---

## What to do next

Pick **one** action this week:

| If you're at Tier… | Do this |
|---|---|
| 1–2 | Load local setup (Phase 17); browse `/tools` and `/resources` |
| 2–3 | Run Phase 19 experiment |
| 3 | Open a small PR (Phase 22 suggested first PRs) |
| 3+ data interest | File one structured GitHub issue or request contributor access |
| 4+ | Comment on an open issue offering to implement with your proposed approach |

---

## Onboarding complete

This series was **read-only investigation** unless you ran local experiments yourself. You now have:

- A **mental model** of the factory (Phases 1–6)
- **Pipeline depth** for CMS, import, export, audio, mushaf, morphology (Phases 8–14)
- **Operational knowledge** of jobs, frontend, setup, URLs (Phases 15–18)
- **Practice and quality** norms (Phases 19–22)
- A **map** of where to plug in (this phase)

You're not expected to know every model or rake task. You're expected to know **where to look**, **what not to break**, and **how to ask**.

Welcome to the factory floor.

---

## Optional: upstreaming this series

The `onboarding/` folder is currently **local/untracked** in many clones. If Tarteel wants it in the main repo:

1. Ask in an issue whether `docs/onboarding/` or `app/views/docs/markdown/onboarding-*.md` is preferred
2. Open a dedicated PR with only onboarding files
3. Link from `contribute-code.md` or README

**INFERENCE** — Don't bundle 23 files into an unrelated feature PR.

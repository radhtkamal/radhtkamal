# Phase 20 — Testing & Data Integrity

> **Onboarding series:** progressive deep-dive for becoming a legitimate QUL contributor.  
> **Prerequisites:** [Phases 1–19](phase-01-what-is-qul.md)  
> **This file:** how QUL verifies code and data quality — automated tests, linters, CMS integrity tools, and what is *not* gated in CI.

---

## The honest headline

QUL quality control is **layered but uneven**:

| Layer | Exists? | Blocks merge in CI? |
|---|---|---|
| Minitest unit tests (`test/`) | Yes — **~17 files**, mostly services | **No** — no test workflow in `.github/workflows/` |
| RuboCop (Ruby style) | Yes | **No** — not in CI workflows observed |
| ESLint (JS/Vue) | Yes (`package.json`) | **No** |
| CodeQL security scan | Yes | Yes (on `main` PRs) |
| CMS data integrity checks | Yes — ~25 admin diagnostics | Manual — maintainer runs in `/cms` |
| Import/post-approve hooks | Yes — reports issues, rarely blocks | No automatic block |
| Contributor manual testing | Expected | PR template asks for it |

**INFERENCE:** For OSS contributors, **your PR description and local verification matter more than a green CI test badge**. This is typical for data-heavy Rails apps with thin automated coverage.

---

## Automated tests (Minitest)

### Framework and location

| Item | Detail |
|---|---|
| Framework | **Minitest** (Rails default) — no `spec/` directory |
| Test root | `test/` |
| Helper | `test/test_helper.rb` |
| Run all | `bin/rails test` |
| Run one file | `bin/rails test test/services/audio/segment_validator_test.rb` |

**FACT** — There is **no `group :test` block** in `Gemfile`; Minitest ships via Rails/Bundler (`minitest` in `Gemfile.lock`).

### What is actually tested

Current test files cluster around **pure logic** that does not need a full Quran database:

| Area | Example test file |
|---|---|
| Resource search | `test/services/resources/search_query_test.rb` |
| Quran reference parsing | `test/services/resources/quran_reference_parser_test.rb` |
| Arabic search/normalize | `test/services/search/arabic_normalizer_test.rb` |
| Audio segment validation | `test/services/audio/segment_validator_test.rb` |
| Morphology treebank | `test/services/morphology/treebank/*_test.rb` |
| Corpus POS tags | `test/lib/corpus/pos_tags_locale_test.rb` |

### Test style: stubbed models, not full Rails

**FACT** — `test/test_helper.rb` defines **fake** `Chapter`, `Verse`, and `ResourceContent` constants when missing — tests run without loading `quran_dev`.

```ruby
# test/test_helper.rb pattern
Chapter.records = [FakeChapterRecord.new(...)]
Verse.records = [FakeVerseRecord.new(verse_key: '2:255', ...)]
```

**INFERENCE:** This is intentional — tests are fast and do not require the mini dump. It also means **integration paths** (controllers, Active Record across two DBs, export jobs) are largely **untested** in CI.

### When to add a test

| Change type | Test expectation |
|---|---|
| New/changed service in `app/services/` | **Strongly encouraged** — follow neighbor files in `test/services/` |
| Pure lib logic (`lib/corpus/`, parsers) | Add `test/lib/...` |
| Controller-only glue | Manual browser test often accepted |
| CMS admin button wiring | Manual |
| Data migration / Quran content | Integrity checks + manual spot verify |

**Example:** If you fix `Audio::SegmentValidator`, extend `test/services/audio/segment_validator_test.rb` with a regression case (the file already documents a real recitation bug).

---

## Linters and formatters

### Ruby — RuboCop

```bash
bundle exec rubocop
bundle exec rubocop -a path/to/file.rb   # auto-correct safe cops
```

**FACT** — `.rubocop.yml` targets Ruby 3.3, Rails 8.0; includes `rubocop-rails`, `rubocop-performance`, `rubocop-minitest`.

Excludes: `db/schema.rb`, `db/migrate/**`, `bin/**`, `config/environments/*`.

### JavaScript / Vue — ESLint

```bash
yarn lint
yarn lint:fix
```

**FACT** — `package.json` `lint-staged` runs Prettier + ESLint on `app/**/*.{js,jsx}` and ERB lint on views.

### ERB lint

`lint-staged` references `bundle exec erblint` for `app/views/**/*.html.erb`, but **erblint is not in `Gemfile`** as of this survey.

**INFERENCE** — Pre-commit ERB lint may fail locally unless erblint is installed separately. Not a CI gate.

---

## CI workflows (what runs on GitHub)

| Workflow | File | Purpose |
|---|---|---|
| CodeQL | `.github/workflows/codeql.yml` | Security analysis on `main` push/PR |
| Deploy | `.github/workflows/deploy.yml` | Production deploy (not a test gate for contributors) |

**FACT** — No `rails test` or `rubocop` job in `.github/workflows/`.

**INFERENCE** — Maintainers rely on review + manual verification. Do not assume CI will catch a broken test you didn't run locally.

---

## E2E tests (Cypress)

**FACT** — Cypress lives in a **separate package**: `scripts/cypress-e2e/`

| Spec | Covers |
|---|---|
| `signinTests.cy.js` | Devise sign-in |
| `signupTests.cy.js` | Registration |

```bash
cd scripts/cypress-e2e
npx cypress open    # interactive
npx cypress run     # headless
```

**INFERENCE** — E2E coverage is minimal (auth only). Contributor tools, CMS, exports are **not** covered by Cypress in this repo.

Root `package.json` lists `cypress` as devDependency but tests run from the subfolder.

---

## Data integrity — the real Quran QA system

For Quranic **data** quality, QUL invests in **admin diagnostics** more than automated tests.

### CMS dashboard entry point

**URL:** http://localhost:3000/cms (dashboard)

**FACT** — Panel **"Data Integrity checks"** lists all checks from `Tools::DataIntegrityChecks.checks` with links to:

```
/cms/data_integrity_check?check_name=<check_name>
```

Implementation:

| Piece | Path |
|---|---|
| Check definitions | `app/models/tools/data_integrity_checks.rb` (~1200 lines) |
| Admin page | `app/admin/tools/data_integrity_check.rb` |
| Tajweed-specific | `app/models/tools/tajweed_rules_check.rb` |

**Note:** Tajweed panel on dashboard is wrapped in `if false` — checks still work via direct URL if registered.

### Check catalog (representative)

| `check_name` | What it finds |
|---|---|
| `words_without_root` | Morphology gaps |
| `words_without_lemma` | Lemma assignment missing |
| `words_without_stem` | Stem assignment missing |
| `words_with_missing_arabic_text` | Empty script columns |
| `ayah_with_missing_translations` | Translation coverage holes |
| `words_with_missing_translations` | Word translation gaps |
| `ayah_with_missing_tafsirs` | Tafsir coverage holes |
| `compare_translations` | Discrepancies between editions |
| `duplicate_mushaf_words` | Duplicate layout rows |
| `mushaf_words_with_incorrect_position` | Line/page position errors |
| `ayah_with_different_mushaf_page` | Same ayah on different pages across mushafs |
| `compare_two_mushaf_words` | Word-level mushaf diff |
| `ayah_without_matching_ayahs` | Similar-ayah reference gaps |

Each check returns a **paginated table** with optional filter fields (mushaf selectors, resource ids) and deep links into CMS preview pages.

**FACT** — These are **read-only diagnostics**. They do not auto-fix data or block exports.

### How checks are structured

```ruby
def self.words_without_root
  {
    name: "...",
    description: "...",
    instructions: ["..."],
    table_attrs: ['verse_key', 'word_id', ...],
    fields: [{ type: :select, name: :mushaf_id, ... }],
    links_proc: { verse_key: ->(record, _) { [record.verse_key, "/cms/verses/..."] } },
    check: ->(params) { /* SQL / AR query returning rows */ }
  }
end
```

**INFERENCE** — Adding a new integrity check = new class method + entry in `.checks` array. No migration needed.

---

## Import and approve hooks (soft validation)

When content is imported or approved, QUL runs **post-hooks** that collect issues:

```ruby
# ResourceContent#run_after_import_hooks (after approve job)
if translation?
  check_for_missing_translation   # expects 6236 rows, non-blank text
elsif tafsir?
  check_for_missing_tafsirs
end
```

```ruby
# DraftContent::ApproveDraftContentJob
issues = @resource.run_after_import_hooks
report_issues(issues)  # → ActiveAdmin::Comment on ResourceContent
```

**FACT** — Issues are **reported as CMS comments**, not raised as exceptions. Approve can "succeed" with data gaps.

**INFERENCE** — Maintainers review comments before `refresh_export!`.

---

## Domain-specific validators

### Audio segments

| Component | Role |
|---|---|
| `Audio::SegmentValidator` | `app/services/audio/segment_validator.rb` |
| Unit tests | `test/services/audio/segment_validator_test.rb` |
| Contributor UI | `POST .../validate_segments` on surah audio tool |
| CMS | "Validate segments" action on recitation admin pages |

Validates overlaps, gaps, word count mismatches, duration bounds — categories returned as structured issues.

### Mushaf / translation

No dedicated validator classes with test suites. Quality relies on integrity SQL checks + human proofreading tools.

---

## Manual verification playbook (for PRs)

Use this checklist based on what you changed:

### Code-only (no Quran data)

```text
□ bin/rails test (or targeted test file)
□ bundle exec rubocop on changed Ruby files
□ yarn lint if JS/Vue changed
□ bin/dev smoke: load affected page in browser
```

### Translation / tafsir data

```text
□ Phase 19 experiment: draft row created, published unchanged
□ /cms/draft_translations filter by resource_content_id
□ After approve (if applicable): check_for_missing_translation issues in CMS comments
□ Spot-check /ayah/:key in browser
□ Optional: ayah_with_missing_translations integrity check
```

### Mushaf layout

```text
□ Save one page locally, verify MushafWord rows in console
□ duplicate_mushaf_words check
□ mushaf_words_with_incorrect_position check
□ Export preview if you touched exporter
```

### Audio segments

```text
□ validate_segments in contributor UI
□ bin/rails test test/services/audio/segment_validator_test.rb
□ Spot-listen one ayah in segment builder
```

### Export / downloader

```text
□ refresh_export! in CMS (needs Sidekiq + S3 or expect failure)
□ Download JSON/SQLite locally, verify verse_key keys join to your app
□ Check file size and row counts vs integrity check stats
```

---

## PR expectations

**FACT** — `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md` includes:

> **How Has This Been Tested?** — describe environment, commands, scenarios.

**FACT** — `best-practices.md` recommends focused PRs and verification scripts for cross-resource joins.

**INFERENCE** — Write explicit manual steps reviewers can replay. "Tested locally" without commands is weak for this repo.

Good PR testing section example:

```markdown
## How Has This Been Tested?
- `bin/rails test test/services/audio/segment_validator_test.rb` — pass
- `bundle exec rubocop app/services/audio/segment_validator.rb` — pass
- Loaded `/surah_audio_files/1/segment_builder?recitation_id=7`, ran Validate — 0 errors
- Ran CMS check `ayah_with_missing_translations` for resource 131 — no new gaps
```

---

## Testing pyramid (QUL reality)

```text
                    ┌─────────────────┐
                    │ Manual CMS +    │  ← primary for data
                    │ integrity checks│
                    └────────┬────────┘
              ┌──────────────┴──────────────┐
              │  Minitest (services/lib)   │  ← growing, isolated
              └──────────────┬──────────────┘
        ┌────────────────────┴────────────────────┐
        │  Cypress (auth only, optional/local)   │
        └────────────────────┬────────────────────┘
  ┌──────────────────────────┴──────────────────────────┐
  │  CodeQL security scan (CI)                         │
  └───────────────────────────────────────────────────┘
```

---

## What is NOT tested (gaps to know)

| Gap | Risk | Mitigation |
|---|---|---|
| No controller/request tests | Routing/param bugs | Manual URL testing (Phase 18) |
| No job integration tests | Approve/export failures | Run Sidekiq locally, watch logs |
| No two-DB transaction tests | Cross-DB inconsistency | Console verification |
| No export golden-file tests | Format regressions | Download and diff export |
| Quran models untested in CI | AR callback surprises | Integrity checks + spot console queries |
| `segment_pipeline` routes | 404 / missing code | Avoid until controller lands |

**Do not assume** high coverage because the app is production-grade. The production grade comes from **operational discipline** (maintainers + integrity tooling + curated dumps).

---

## Suggested first test contribution

If you want a low-risk OSS code contribution that fits the repo's style:

1. Find a bug or edge case in an existing service that already has tests (search, segment validator, treebank).
2. Add one Minitest case reproducing it.
3. Fix the service.
4. `bin/rails test <that file>` in PR description.

This matches existing conventions better than adding the project's first controller spec.

---

## Uncertainties

| # | Question | Label |
|---|---|---|
| 1 | Whether maintainers run full `bin/rails test` before merge | **INFERENCE** — yes informally; not enforced by CI |
| 2 | Whether erblint is required locally | **UNKNOWN** — in lint-staged, not in Gemfile |
| 3 | Planned CI test workflow | **UNKNOWN** — none today |
| 4 | Full count of integrity checks (tajweed + data) | **FACT** — 26 in `DataIntegrityChecks.checks` + 4 in `TajweedRulesCheck` |

---

## Phase 20 summary

```text
bin/rails test     → fast service/unit tests (stubbed Quran models)
bundle exec rubocop → Ruby style (local)
/cms dashboard     → Data Integrity checks (real Quran DB queries)
approve hooks      → soft validation → Admin comments
CI                 → CodeQL only; you own manual verification in PRs
```

For QUL, **data integrity tooling in CMS is as important as the test folder**. Learn both before shipping content or export changes.

---

## Stop here — questions before Phase 21

Phase 21 covers **engineering culture and PR expectations** — how Tarteel/QUL reviewers think about risk, scope, and contributor trust.

1. Does CI run `bin/rails test` on your PRs?
2. Where do you run `words_without_root` locally?
3. What happens when `check_for_missing_translation` finds gaps during approve — does the job fail?

Reply with questions, or say **"proceed"** for **Phase 21 — Engineering Culture & PR Expectations**.

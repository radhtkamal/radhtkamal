# Phase 21 — Engineering Culture & PR Expectations

> **Onboarding series:** progressive deep-dive for becoming a legitimate QUL contributor.  
> **Prerequisites:** [Phases 1–20](phase-01-what-is-qul.md)  
> **This file:** how QUL/Tarteel thinks about contributions — risk tiers, review norms, issue etiquette, and what makes a PR easy to merge.

---

## Two legitimate contribution paths

QUL deliberately separates **platform code** from **Quranic data**:

| Path | You change | Typical entry | Review focus |
|---|---|---|---|
| **Code** | Rails app, exporters, UI, docs site | Fork → PR on GitHub | Correctness, scope, regressions |
| **Data** | Translations, audio segments, mushaf, morphology | CMS + proofreading tools | Accuracy, provenance, join keys |

**FACT** — `contribute-code.md` and `contribute-data.md` are separate docs on `/docs`.

**INFERENCE:** A PR that mixes a large data migration with a UI refactor will be hard to review. Split them.

---

## Cultural values (inferred from docs + repo behavior)

These are not written as a manifesto, but show up consistently:

### 1. Published data is sacred

- Contributor tools write **drafts** first (Phase 9).
- CMS **approve** promotes to Quran DB.
- **Export** is a separate, deliberate step (Phase 11).
- Integrity checks exist because bad data ships to thousands of apps via `/resources`.

**Reviewer mindset:** "What happens to downstream consumers if this is wrong?"

### 2. Identifiers must stay stable

`verse_key`, `location`, `word_position`, `resource_content_id` are join contracts (Phases 1–2).

**High-risk changes:** Renaming keys in exports, changing cardinality, altering `verse_index` semantics.

### 3. Scope beats ambition

**FACT** — `best-practices.md`: *"Keep pull requests focused by concern (docs, integrations, feature)."*

Recent merged PRs are small and descriptive:

```text
Fix typo in quran-script group description (#745)
fix: paginate word mistakes by per_page instead of hardcoded 100 (#721)
docs: clarify that audio_url points at Tarteel's CDN (#693)
```

**INFERENCE:** One logical change per PR. Typos and focused fixes merge quickly.

### 4. Docs are product

Setup gaps should be fixed via PR (`project-setup.md` explicitly invites this).

**Stale path alert:** `contributing.md` says edit `docs/` — **wrong**. Canonical docs live in `app/views/docs/markdown/` + `config/docs.yml` (Phase 1, 18).

### 5. Community coordination for large data work

**FACT** — `contribute-data.md` points to [Discord](https://t.zip/discord?utm_source=github&utm_campaign=qul) for coordinating larger data efforts.

**INFERENCE:** Don't silently re-segment a full recitation or re-import a translation without maintainer alignment.

---

## Risk tiers (how reviewers classify your change)

Use this to self-triage before opening a PR:

| Tier | Examples | Review bar | Your job |
|---|---|---|---|
| **Low** | Typo, CSS, doc fix, test for pure service | Light | Show command output |
| **Medium** | New Stimulus behavior, search tweak, exporter field | Medium | Tests + manual URL |
| **High** | Approve job, import pipeline, export format | Heavy | Tests, integrity check, sample export diff |
| **Critical** | Quran schema, bulk translation replace, mushaf mass edit | Maintainer-led | Issue first, phased PRs |

**INFERENCE:** Critical-tier work without an issue will likely stall.

---

## Issues before PRs

**FACT** — PR template states:

> *"This project only accepts pull requests related to open issues"*  
> *"If suggesting a new feature or change, please discuss it in an issue first"*

**Practical interpretation:**

| Situation | Action |
|---|---|
| Obvious typo / docs command fix | PR ok, issue optional |
| New feature or behavior change | Open issue first |
| Data correction in published resource | Use structured issue template |
| Want CMS / project access | Contributor application issue |

---

## GitHub issue templates (data culture)

Structured templates encode **what maintainers need**:

| Template | Labels | Notable requirement |
|---|---|---|
| Translation issue | `content issue`, `translation` | Factual errors only — not wording preference |
| Quran script / font | `content issue`, `script` | Surah, ayah, script type |
| Mushaf layout | `layout issue` | Layout name + page number |
| New mushaf layout | `mushaf layout` | PDF URL, page count, lines per page |
| Contributor application | `contributor-request` | Volunteer OSS license acknowledgment |

**FACT** — Translation template explicitly says:

> *"This report is for typos, spelling mistakes, and factual issues only. Please do not use this form to suggest changes in wording preferences or subjective opinions."*

**INFERENCE:** QUL treats translations as **authoritative editions**, not crowd-rewritten paraphrases. Disputes about theology or style go through proper scholarly channels, not drive-by PRs.

**FACT** — Several content templates auto-assign `naveed-ahmad` — **INFERENCE:** core maintainer for content triage.

---

## Getting contributor access (CMS / projects)

Code contributors can run the app locally with `db:seed` super-admin (Phase 17).

**Production-style data contribution** requires:

1. **GitHub account** + proofreading tools (some edits need login)
2. **`UserProject`** approval for a specific `ResourceContent` — `can_manage?` checks this (Phase 18)
3. Optional: **Contributor Application** issue (`.github/ISSUE_TEMPLATE/contributor-application.yml`) for volunteer coordination

```yaml
# Contributor application asks:
# - How would you like to contribute?
# - Agreement: volunteer, open-source license
```

**FACT** — `users.approved` column exists on User model — **INFERENCE:** registration may require admin approval for full access (exact gate not fully traced; super_admin bypasses project checks).

---

## PR anatomy (what reviewers look for)

### Title

Patterns from recent history:

| Style | Example |
|---|---|
| Imperative fix | `Fix typo in quran-script group description` |
| Conventional prefix | `fix:`, `docs:`, `feat:` (used sometimes, not mandatory) |
| Descriptive feature | `Add audio repeat and use Digitalkhatt script in segments` |

**Guideline:** Reader should know **what** changed without opening the diff.

### Description (use the template)

```markdown
## Description
## Related Issue        ← link #123
## Motivation and Context
## How Has This Been Tested?   ← Phase 20 playbook
## Screenshots (if appropriate)
```

**Weak:** "Tested locally"  
**Strong:** Commands run, URLs clicked, integrity check name, before/after export snippet

### Diff scope

| Good | Avoid |
|---|---|
| One feature or fix | Drive-by refactors in same PR |
| Matching existing style | New abstraction for one call site |
| Tests when touching `app/services/` | Changing export JSON shape without migration note |

### Screenshots

Expected for: UI changes, CMS admin buttons, Arabic rendering, segment builder, mushaf preview.

---

## Code review themes (what gets comments)

Based on architecture across onboarding phases:

### Data safety

- Does this write directly to `translations` / `Audio::Segment` when it should use drafts?
- Does it disable PaperTrail during bulk writes without restoring?
- Does it assume `verses.id` when the UI uses `verse_key`?

### Performance

- `find_each` / `insert_all` for bulk — not `each` + `save` on 6236 rows
- N+1 queries in admin index pages
- Export loading entire tables into memory

### Export compatibility

- Are `verse_key` / `location` keys preserved in JSON?
- Will existing downloaders break if field names change?
- Is CDN hotlinking documented if adding `audio_url` examples?

### Frontend

- Stimulus vs new Vue island — default Stimulus (Phase 16)
- `data-turbo="false"` when jQuery widgets break
- Arabic font classes (`qpc-hafs`, etc.) for script preview

### Docs

- Edit `app/views/docs/markdown/`, update `config/docs.yml` if adding pages
- Re-run setup commands you changed
- Note consumer-facing impact in PR body

---

## What tends to get rejected or delayed

| Change | Why |
|---|---|
| Large unsolicited translation rewording | Scholarly / licensing — use issue template |
| Export format break without versioning | Downstream apps depend on stable shapes |
| Direct Quran DB mass update in contributor path | Bypasses draft/review |
| PR without issue for new feature | Template explicitly asks for issue |
| Mixing unrelated refactors | Review fatigue |
| Depends on missing code (`segment_pipeline` controller) | Broken routes |
| Production CDN hotlinking in docs without self-host warning | Policy (see recitation tutorial PR #693) |

---

## Licensing and attribution

**FACT** — Contributor application checkbox:

> *"I understand that this is a volunteer-based project and my contributions will be used under an open-source license."*

**INFERENCE:** Code and data contributions are OSS — check repo `LICENSE` file for exact terms before contributing substantial work.

**FACT** — Resources carry provenance metadata (`ResourceContent`, `DataSource`, author fields) — data PRs should respect source licenses and attribution chains (Phase 3).

---

## Communication channels

| Channel | Use for |
|---|---|
| **GitHub Issues** | Bugs, data reports, feature discussion |
| **GitHub PRs** | Code and docs changes |
| **Discord** | Coordinating large data work, asking process questions |
| **CMS comments / AdminTodo** | Internal maintainer workflow — not for external contributors initially |

**INFERENCE:** Public technical discussion prefers GitHub for traceability; Discord for real-time coordination.

---

## Maintainer trust ladder

```text
1. Issue or small doc PR           → proves communication + setup
2. Service test + bugfix PR        → proves code conventions
3. Proofreading suggestions        → proves data care (drafts only)
4. UserProject / contributor app   → scoped CMS access
5. Export / import pipeline work   → maintainer partnership
```

You do not need step 5 to be a **valuable** OSS contributor. Doc fixes, segment validator tests, and integrity check improvements are real contributions.

---

## Self-review checklist (before "Create pull request")

```text
□ One concern per PR
□ Issue linked (if feature/non-trivial)
□ bin/rails test (relevant files) — Phase 20
□ bundle exec rubocop (changed Ruby)
□ Manual steps documented in PR template
□ No secrets (.env, credentials, master.key)
□ Docs path correct (app/views/docs/markdown/)
□ Data changes use draft path (not direct publish)
□ Export backward compatibility considered
□ Screenshots for UI
```

---

## Example PR descriptions (templates you can copy)

### Doc fix

```markdown
## Description
Fix setup instructions: `bin/setup` does not load the Quran mini dump.

## Related Issue
Fixes #___

## How Has This Been Tested?
- Followed updated steps on macOS + Postgres 16
- `Verse.count` > 0 after `psql -d quran_dev -f mini_quran_dev.sql`
- `/ayah/2:255` loads
```

### Service bugfix

```markdown
## Description
Segment validator: flag ayah overlap when timestamp_to > next timestamp_from.

## Related Issue
Fixes #___

## How Has This Been Tested?
- `bin/rails test test/services/audio/segment_validator_test.rb` — 42 runs, 0 failures
- Reproduced case from recitation 1 surah 32 in segment builder validate UI
```

### UI tweak

```markdown
## Description
Change proofreading submit button label from "Purpose changes" to "Propose changes".

## How Has This Been Tested?
- Screenshot attached
- Submitted test draft on local translation_proofreadings — redirect + flash ok
```

---

## Uncertainties

| # | Question | Label |
|---|---|---|
| 1 | Formal CODEOWNERS or required reviewers | **FACT** — none in repo |
| 2 | Whether all new users need `approved: true` before login | **UNKNOWN** — column exists |
| 3 | SLA for PR review / issue response | **UNKNOWN** — volunteer project |
| 4 | Whether `contributing.md` `docs/` path will be fixed upstream | **UNKNOWN** — stale as of this survey |

---

## Phase 21 summary

```text
Code PRs     → focused, tested, issue-linked when non-trivial
Data issues  → structured templates, factual corrections, identifiers included
Data edits   → drafts → CMS review → export (never skip layers)
Culture      → stable joins, provenance, small PRs, docs as product
Access       → UserProject / contributor application for scoped CMS work
```

Reviewers are guarding **downstream Quran apps**, not just Rails aesthetics. Frame your PR around consumer impact and review becomes collaborative.

---

## Stop here — questions before Phase 22

Phase 22 covers **OSS git workflow** — branching, rebasing on upstream, keeping forks current, and PR hygiene mechanics.

1. When should you open an issue before a PR?
2. What's the difference between a code contribution and a data contribution path?
3. Why does the translation issue template reject "wording preference" reports?

Reply with questions, or say **"proceed"** for **Phase 22 — OSS Git Workflow**.

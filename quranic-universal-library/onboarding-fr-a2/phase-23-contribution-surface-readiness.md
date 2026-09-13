# Phase 23 — Où contribuer et es-tu prêt ?

> **Série d'intégration :** on apprend QUL étape par étape.  
> **Avant :** [Phases 1–22](phase-01-what-is-qul.md)  
> **Ce fichier :** où tu peux réellement aider, selon tes compétences. Plus une liste de contrôle honnête pour savoir quand tu es prêt.

---

## Tu es ici

Si tu as lu les Phases 1–22, tu as un **modèle mental fonctionnel** de QUL :

```text
Usine (Rails CMS + outils)  →  Quran DB curée  →  zips exportés sur /resources
         ↑                              ↑
    deux Postgres DBs              jointures verse_key / location
    édition draft-first              vérifications d'intégrité dans /cms
```

Cette phase finale transforme cette connaissance en **points d'entrée actionnables** et une **auto-évaluation**.

---

## Carte des surfaces de contribution (par compétence)

### Profil A — Ingénieur React / TypeScript / Node

Meilleures correspondances : **outillage frontend, consommateurs API, tests de services, docs, petit glue Rails**. Pas ActiveRecord profond dès le premier jour.

| Surface | URL / chemin | Zone code | Risque | Commencer ici ? |
|---|---|---|---|---|
| **Site docs** | `/docs` | `app/views/docs/markdown/`, `config/docs.yml` | Faible | **Oui** — corriger chemins obsolètes `contributing.md` |
| **UI catalogue ressources** | `/resources` | `ResourcesController`, `app/views/resources/` | Moyen | Après trace Phase 18 |
| **Visionneuse ayah** | `/ayah/2:255` | `AyahController`, Turbo frames, Stimulus | Moyen | Bon second projet |
| **Segment builder (Vue)** | `/surah_audio_files/.../segment_builder` | `app/javascript/segments/` | Moyen–Élevé | Si tu connais l'UX audio |
| **Recherche / parsers** | `/resources?q=…` | `app/services/resources/`, `test/services/` | Moyen | **Oui** — a des tests unitaires |
| **Validateur segment** | contributeur + CMS | `app/services/audio/segment_validator.rb` | Moyen | **Oui** — service testé |
| **Outils Stimulus** | `/tools` | `app/javascript/controllers/` | Faible–Moyen | Bugs UI, a11y |
| **API JSON v1** | `/api/v1/*` | `app/controllers/api/v1/` | Élevé | Issue d'abord — contrat public |
| **Exporteurs** | Boutons export CMS | `lib/exporter/` | Élevé | Binôme mainteneur |
| **Pipelines import** | Import CMS | `lib/importer/` | Critique | Pas en solo tôt |
| **Active Admin** | `/cms` | `app/admin/` | Moyen | Après flux CMS (Phase 8) |

**On pense :** Ton chemin le plus rapide vers une PR mergée est **docs + tests de services + Stimulus**. Domaines qui correspondent à ta stack et ont des critères de revue clairs.

---

### Profil B — Contributeur données / contenu

Meilleures correspondances : **relecture, segmentation audio, mushaf, annotation morphologie**. Ruby minimal requis.

| Surface | URL outil | Écrit vers | Accès requis |
|---|---|---|---|
| Relecture traduction | `/translation_proofreadings` | `draft_translations` (CMS) | Login + `UserProject` ou super_admin |
| Relecture tafsir | `/tafsir_proofreadings` | draft tafsirs | Même |
| Traduction mot | `/word_translations` | draft traductions mot | Même |
| Segments audio sourate | `/surah_audio_files` | `audio_segments` (Quran) | Annotateur audio ou projet |
| Segments audio ayah | `/ayah_audio_files` | `audio_files.segments` | Même |
| Dispositions mushaf | `/mushaf_layouts` | `mushaf_words` (Quran) | Accès projet |
| Annotation tajweed | `/tajweed_words` | tables tajweed | Accès projet |
| Mutashabihat | `/morphology_phrases` | phrases morphologie | Accès projet |
| Graphes de dépendance | `/morphology/dependency-graphs` | nœuds/arêtes graphe | Accès projet |
| Concordance mot | `/word_concordance_labels` | labels morphologie | Accès projet |
| Relecture script | `/word_text_proofreadings` | texte script | Accès projet |

**Signaler des issues sans CMS :** Templates d'issues GitHub (traduction, mushaf, script).

**Obtenir l'accès :** Issue candidature contributeur + Discord pour les gros efforts (Phase 21).

---

### Profil C — Ingénieur Rails / backend

| Surface | Chemin | Notes |
|---|---|---|
| Jobs Sidekiq | `app/jobs/` | Approve, export, import — Phase 15 |
| Modèles / concerns | `app/models/` | Connaître `ApplicationRecord` vs `QuranApiRecord` |
| Importeurs | `lib/importer/` | Phase 10 |
| Exporteurs | `lib/exporter/` | Phase 11 |
| Tâches Rake | `lib/tasks/` | Ops données ponctuelles |
| Actions CMS | `app/admin/` | Câbler boutons → jobs |
| Vérifications d'intégrité | `app/models/tools/data_integrity_checks.rb` | Ajouter diagnostics SQL |

---

### Profil D — DevOps / plateforme

| Surface | Chemin | Notes |
|---|---|---|
| Docker / deploy | `Dockerfile`, `.github/workflows/deploy.yml` | Image production |
| CodeQL | `.github/workflows/codeql.yml` | Seule barrière CI aujourd'hui |
| Sidekiq / Redis | `config/sidekiq.yml`, `docker-compose.yml` | Ops workers |
| Stockage S3 | `config/storage.yml`, `.env.sample` | Uploads export |
| Sauvegardes DB | `BackupJob`, `lib/utils/db_backup.rb` | Planifié |

**Opportunité de lacune :** Ajouter un job CI pour `bin/rails test` + `rubocop`. Aiderait tous les contributeurs (Phase 20).

---

## Carte code → phases d'intégration

Référence rapide quand tu es perdu :

| Si tu travailles sur… | Relire |
|---|---|
| Ce qu'est QUL | [Phase 1](phase-01-what-is-qul.md) |
| `verse_key`, `ResourceContent` | [Phase 2](phase-02-quranic-data-model.md) |
| Drafts, PaperTrail | [Phase 3](phase-03-provenance-and-integrity.md) |
| Bases Rails | [Phase 4](phase-04-rails-for-react-node-engineers.md) |
| Deux bases de données | [Phase 5](phase-05-two-database-architecture.md) |
| Schéma système | [Phase 6](phase-06-architecture-mental-model.md) |
| Structure dossiers | [Phase 7](phase-07-repository-tour.md) |
| Flux approve CMS | [Phase 8](phase-08-cms-flow.md) |
| Chemin édition contributeur | [Phase 9](phase-09-core-runtime-flow.md) |
| Imports | [Phase 10](phase-10-import-pipeline.md) |
| Exports / `/resources` | [Phase 11](phase-11-export-pipeline.md) |
| Morphologie | [Phase 12](phase-12-morphology-arabic-linguistic-data.md) |
| Mushaf | [Phase 13](phase-13-mushaf-layouts.md) |
| Audio | [Phase 14](phase-14-audio-subsystem.md) |
| Sidekiq | [Phase 15](phase-15-background-jobs-sidekiq.md) |
| Frontend | [Phase 16](phase-16-frontend-reality.md) |
| Setup local | [Phase 17](phase-17-local-setup.md) |
| URL → code | [Phase 18](phase-18-browser-code-correlation.md) |
| Expérience pratique | [Phase 19](phase-19-controlled-learning-experiment.md) |
| Tests / intégrité | [Phase 20](phase-20-testing-and-data-integrity.md) |
| Culture / PRs | [Phase 21](phase-21-engineering-culture-pr-expectations.md) |
| Workflow git | [Phase 22](phase-22-oss-git-workflow.md) |

---

## Niveaux de préparation (auto-évaluation)

Coche chaque case. **Niveau 3 = prêt pour des PR code significatives.** Niveau 4 = prêt pour le travail pipeline données.

### Niveau 1 — Littératie consommateur (pas de setup local)

- [ ] Je peux expliquer QUL vs téléchargements `/resources`
- [ ] Je connais le format `verse_key` (`"2:255"`)
- [ ] Je comprends que les traductions sont des packages indexés par `resource_content_id`
- [ ] Je ne ferai pas de hotlink `audio-cdn.tarteel.ai` dans des apps production

**Tu peux :** Utiliser les données QUL dans tes apps. Signaler des problèmes via les templates GitHub.

---

### Niveau 2 — Observateur local

- [ ] `bin/setup` + mini dump chargé (`Verse.count > 0`)
- [ ] `bin/dev` tourne ; `/ayah/2:255` charge
- [ ] Je peux me connecter à `/cms` avec l'admin seed
- [ ] J'ai tracé une URL vers contrôleur + vue (Phase 18)

**Tu peux :** Explorer en local. Corriger la doc. Ouvrir des issues informées.

---

### Niveau 3 — Contributeur code (minimum recommandé)

- [ ] Expérience Phase 19 complétée OU trace draft équivalente
- [ ] Je connais les emplacements tables CMS DB vs Quran DB
- [ ] J'ai exécuté `bin/rails test` sur un fichier pertinent
- [ ] Je peux ouvrir une PR ciblée avec étapes de test (Phases 20–22)
- [ ] Je sais ne pas éditer `docs/` racine pour la doc site web

**Tu peux :** PRs docs, tests de services, fixes Stimulus, améliorations search/parser.

---

### Niveau 4 — Contributeur pipeline données

- [ ] Niveau 3 complet
- [ ] Sidekiq + Redis tourne en local
- [ ] Tracé import OU export une fois dans les logs (Phases 10–11)
- [ ] Exécuté au moins une vérification d'intégrité CMS
- [ ] Coordonné avec les mainteneurs (issue/Discord) pour données bulk

**Tu peux :** Travail approve/import/export avec revue. Édition CMS ciblée.

---

### Niveau 5 — Profil mainteneur (aspirationnel)

- [ ] Niveau 4 complet
- [ ] À l'aise pour lire `lib/importer/` et `lib/exporter/` de bout en bout
- [ ] Comprend `segment_locked`, règles skip export, anomalies cross-DB
- [ ] Peut ajouter une vérification d'intégrité ou extension job approve en sécurité

**Tu peux :** Posséder le cycle de vie complet d'un type de ressource.

---

## Premières contributions recommandées (par niveau)

### Niveau 2 → 3 (code)

| PR | Fichiers | Pourquoi c'est bien |
|---|---|---|
| Corriger chemin docs `contributing.md` | `app/views/docs/markdown/contributing.md` | Haute valeur, risque zéro |
| Corriger bouton "Purpose changes" | `translation_proofreadings/edit.html.erb` | Une ligne, visible utilisateur |
| Ajouter cas test validateur segment | `test/services/audio/segment_validator_test.rb` | Correspond au style test du dépôt |
| Améliorer `project-setup.md` | `app/views/docs/markdown/project-setup.md` | Tu as eu la douleur setup → corrige-la |

### Niveau 3 → 4 (code + conscience données)

| PR | Domaine |
|---|---|
| Améliorer cas limite recherche ressources | `app/services/resources/search_query.rb` |
| Fix UX Stimulus sur `/resources` | `app/javascript/controllers/` |
| Ajouter workflow CI pour `rails test` | `.github/workflows/` (discuter en issue d'abord) |
| Nouvelle vérification d'intégrité (SQL) | `app/models/tools/data_integrity_checks.rb` |

### Niveau 2 → 3 (données, pas de Ruby)

| Action | Canal |
|---|---|
| Rapport typo traduction | Template issue traduction GitHub |
| Relire un ayah (draft) | `/translation_proofreadings` après accès |
| Valider segments audio | Segment builder `/surah_audio_files` |

---

## Ce que tu devrais encore apprendre sur le terrain

Aucune série d'intégration ne remplace ceci. Tu l'acquerras via les PR :

| Sujet | Pourquoi ça reste flou |
|---|---|
| Schéma Quran complet | Uniquement dans le dump chargé ; pas dans `schema.rb` |
| Particularités approve par ressource | Groupes tafsir, notes de bas de page, traductions mot diffèrent |
| Config S3 / CDN production | `.env` + connaissance mainteneur |
| Routes `segment_pipeline` | Contrôleur manquant dans le dépôt |
| Quelles valeurs `resource_content_id` existent dans le mini dump | Sous-ensemble spécifique au dump |
| File de priorité mainteneur | Bande passante bénévole |

**C'est normal.** Les contributeurs légitimes posent des questions dans les issues/PR — avec le contexte de cette série.

---

## Anti-patterns (ne te dis pas prêt si…)

| Signal | Réalité |
|---|---|
| « Je vais corriger la traduction directement en SQL » | Contourne draft → approve → export |
| « Je vais ajouter une app Vue pour chaque outil » | Mauvais pattern frontend (Phase 16) |
| « La CI a passé donc c'est bon » | La CI ne lance pas les tests (Phase 20) |
| « Je vais importer une nouvelle traduction via rake sans issue » | Risque licence + provenance |
| « Je n'ai pas chargé le dump mais je vais éditer les exporteurs » | Impossible de vérifier la sortie |

---

## Index complet des phases

| # | Titre | Fichier |
|---|---|---|
| 1 | Qu'est-ce que QUL ? | `phase-01-what-is-qul.md` |
| 2 | Le modèle de données coranique | `phase-02-quranic-data-model.md` |
| 3 | Provenance et intégrité | `phase-03-provenance-and-integrity.md` |
| 4 | Rails pour ingénieurs React/Node | `phase-04-rails-for-react-node-engineers.md` |
| 5 | Architecture à deux bases de données | `phase-05-two-database-architecture.md` |
| 6 | Modèle mental de l'architecture | `phase-06-architecture-mental-model.md` |
| 7 | Tour du dépôt | `phase-07-repository-tour.md` |
| 8 | Flux CMS | `phase-08-cms-flow.md` |
| 9 | Flux d'exécution principal (édition) | `phase-09-core-runtime-flow.md` |
| 10 | Pipeline d'import | `phase-10-import-pipeline.md` |
| 11 | Pipeline d'export | `phase-11-export-pipeline.md` |
| 12 | Morphologie et données linguistiques arabes | `phase-12-morphology-arabic-linguistic-data.md` |
| 13 | Dispositions mushaf | `phase-13-mushaf-layouts.md` |
| 14 | Sous-système audio | `phase-14-audio-subsystem.md` |
| 15 | Jobs en arrière-plan (Sidekiq) | `phase-15-background-jobs-sidekiq.md` |
| 16 | Réalité du frontend | `phase-16-frontend-reality.md` |
| 17 | Configuration locale | `phase-17-local-setup.md` |
| 18 | Corrélation navigateur ↔ code | `phase-18-browser-code-correlation.md` |
| 19 | Expérience d'apprentissage contrôlée | `phase-19-controlled-learning-experiment.md` |
| 20 | Tests et intégrité des données | `phase-20-testing-and-data-integrity.md` |
| 21 | Culture d'ingénierie et attentes PR | `phase-21-engineering-culture-pr-expectations.md` |
| 22 | Workflow git OSS | `phase-22-oss-git-workflow.md` |
| 23 | **Surfaces de contribution et préparation** | `phase-23-contribution-surface-readiness.md` |

---

## Aide-mémoire une page

```text
PRODUIT     Fichiers exportés sur /resources — pas accès DB live
BASES       quran_community_tarteel (CMS) + quran_dev (Quran)
IDENTITÉ    verse_key, location, resource_content_id
CHEMIN EDIT draft (CMS) → approve (job) → refresh_export! (job) → S3
EXCEPTIONS  mushaf + segments audio écrivent souvent Quran DB directement
DOCS        app/views/docs/markdown/ + config/docs.yml
TESTS       bin/rails test (local) ; vérifications intégrité CMS pour données
PR          fork → upstream/main → petite branche → PR liée issue
PREMIÈRE PR fix doc ou test service ou typo
```

---

## Question finale de préparation

Réponds sans regarder :

1. Où vivent les traductions publiées — CMS ou Quran DB ?
2. Quelles trois commandes amènent un nouveau contributeur du clone à la navigation d'un ayah ?
3. Quelle est la première PR code la plus sûre pour un ingénieur React ?

Si tu as répondu **Quran DB**, **`bin/setup` → charger dump → `bin/dev`**, et **docs ou test service** — tu es prêt à contribuer.

---

## Que faire ensuite

Choisis **une** action cette semaine :

| Si tu es au Niveau… | Fais ceci |
|---|---|
| 1–2 | Charger le setup local (Phase 17) ; parcourir `/tools` et `/resources` |
| 2–3 | Exécuter l'expérience Phase 19 |
| 3 | Ouvrir une petite PR (premières PR suggérées Phase 22) |
| 3+ intérêt données | Déposer une issue GitHub structurée ou demander l'accès contributeur |
| 4+ | Commenter une issue ouverte en proposant d'implémenter |

---

## Intégration terminée

Cette série était une **investigation en lecture seule** sauf si tu as exécuté des expériences locales. Tu as maintenant :

- Un **modèle mental** de l'usine (Phases 1–6)
- **Profondeur pipeline** pour CMS, import, export, audio, mushaf, morphologie (Phases 8–14)
- **Connaissance opérationnelle** des jobs, frontend, setup, URLs (Phases 15–18)
- **Normes pratique et qualité** (Phases 19–22)
- Une **carte** de où tu brancher (cette phase)

On ne t'attend pas pour connaître chaque modèle ou tâche rake. On t'attend pour savoir **où chercher**, **quoi ne pas casser**, et **comment demander**.

Bienvenue sur le plancher de l'usine.

---

## Optionnel : publier cette série upstream

Le dossier `onboarding/` est actuellement **local/non tracké** dans beaucoup de clones. Si Tarteel le veut dans le dépôt principal :

1. Demander dans une issue si `docs/onboarding/` ou `app/views/docs/markdown/onboarding-*.md` est préféré
2. Ouvrir une PR dédiée avec uniquement les fichiers d'intégration
3. Lier depuis `contribute-code.md` ou README

**On pense :** Ne pas bundler 23 fichiers dans une PR fonctionnalité sans lien.

---

*Version A2 — français simple. Généré pendant l'onboarding contributeur QUL.*

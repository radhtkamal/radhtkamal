# Phase 7 — Visite du dépôt

> **Série d'onboarding :** tu apprends QUL étape par étape.  
> **Prérequis :** [Phases 1–6](phase-01-what-is-qul.md)  
> **Ce fichier :** où sont les fichiers dans le dépôt. On suit les couches de la Phase 6.

---

## Comment utiliser cette visite

Ce n'est **pas** une liste de tous les fichiers. C'est une **carte** pour naviguer.

Pour chaque dossier important, on voit :

- **Ce que c'est** (son rôle)
- **Quand tu le touches** (en tant que contributeur)
- **Quoi lire en premier** (par où commencer)

Les chiffres sont approximatifs : ~1 200+ fichiers sous `app/`, ~160 sous `lib/`.

---

## Carte simple du dépôt

```text
quranic-universal-library/
├── app/           ← Application Rails (HTTP, modèles, vues, jobs, admin)
├── lib/           ← Logique métier en masse (import/export/audio/corpus) + tâches rake
├── config/        ← Routes, DB, Sidekiq, manifeste docs, initializers
├── db/            ← Migrations CMS + schema.rb UNIQUEMENT (pas les tables Quran)
├── bin/           ← Wrappers setup, dev, rails
├── docker/        ← Scripts d'init DB dev
├── test/          ← Suite de tests légère (surtout lib/services)
├── scripts/       ← Scripts ops ponctuels (fonts, audio, captures d'écran)
├── public/        ← Assets statiques servis directement
├── onboarding/    ← Cette série (locale, pas encore dans la doc upstream)
├── docs/          ← Presque vide ; PAS la source de vérité de la doc
└── .github/       ← CI (CodeQL, déploiement Docker vers K8s)
```

**Dans le code actuel :** `app/views/docs/markdown/contributing.md` dit *« Edit files in `docs/` first »* — **On sait :** la vraie doc est dans `app/views/docs/markdown/`. Elle est reliée via `DocsManifest` + `config/docs.yml`.

---

## `app/` — la surface Rails (~1 222 fichiers)

Tout ce qui touche l'utilisateur est ici. On peut voir `app/` comme **quatre couches** :

```text
         ┌─────────────────────────────────────┐
         │  views/ + javascript/ + assets/     │  ← ce que voient les utilisateurs
         ├─────────────────────────────────────┤
         │  controllers/ + presenters/ +       │  ← HTTP + mise en forme
         │  finders/ + helpers/ + components/  │
         ├─────────────────────────────────────┤
         │  models/ + services/                │  ← données + petits services
         ├─────────────────────────────────────┤
         │  admin/ + jobs/ + mailers/          │  ← CMS + async + email
         └─────────────────────────────────────┘
```

### `app/controllers/` (~52 fichiers) — entrées HTTP

| Zone | Chemin | Routes à connaître |
|---|---|---|
| **Catalogue public** | `resources_controller.rb`, `landing_controller.rb`, `search_controller.rb` | `/`, `/resources`, `/search` |
| **Docs** | `community_controller.rb` | `/docs`, `/docs/:key`, `/tools` |
| **Outils contributeur** | `translation_proofreadings_controller.rb`, `tafsir_proofreadings_controller.rb`, `mushaf_layouts_controller.rb`, `morphology_phrases_controller.rb`, `word_text_proofreadings_controller.rb`, … | Voir `config/routes.rb` |
| **Outils audio** | `surah_audio_files_controller.rb`, `ayah_audio_files_controller.rb` | Éditeurs de timestamps de segments |
| **Segments (Vue)** | namespace `segments/` | `/segments/*`, `/segment_pipeline/*` |
| **API v1** | `api/v1/` | `/api/v1/chapters`, translations, tafsir, audio |
| **Morphologie publique** | `morphology/` | `/morphology/roots/:id`, lemmas, treebank |
| **Auth** | `users/` (surcharges Devise) | `/users/sign_in`, registrations |
| **Santé** | `health_controller.rb` | `/health` |

**Modèle contributeur :** on trouve l'URL dans `config/routes.rb` → on ouvre le contrôleur → on suit vers presenter/model/service.

**On sait :** `CommunityController#tools` construit la grille d'outils depuis `ToolsHelper#developer_tools`. C'est une liste `ToolCard` codée en dur, pas la base de données.

### `app/admin/` (~113 fichiers) — CMS Active Admin

Organisé par domaine :

| Sous-dossier | Ce que gèrent les admins |
|---|---|
| `content/` | `ResourceContent`, journaux de modifications, infos chapitre, détails racines |
| `quran/` | Versets, mots, pages/words mushaf, tajweed, tokens |
| `audio/` | Récitations, fichiers audio chapitre, jobs métadonnées |
| `draft/` | Traductions brouillon, tafsirs, traductions de mots |
| `downloads/` | `DownloadableResource`, boutons de rafraîchissement export |
| `morphology/` | Grammaire, nœuds de graphe, données de phrases |
| `settings/` | Langues, clients API, tags ressource, types de caractères |
| `tools/` | Vérifications intégrité données, diagnostics |
| `dictionary/`, `grammar/`, `topic.rb`, … | Données de référence support |

**Quand tu le touches :** pour déboguer le CMS, ajouter des champs admin, ou brancher des exports.

**Point d'entrée Phase 8 :** `app/admin/content/resource_content.rb` — le hub approve/import/export.

### `app/models/` (~147 fichiers) — Active Record

Deux classes de base (Phase 5) :

| Base | Exemples |
|---|---|
| `ApplicationRecord` | `User`, `DownloadableResource`, `Draft::Translation`, `UserProject`, `ResourcePermission` |
| `QuranApiRecord` | `Verse`, `Word`, `Translation`, `Tafsir`, `ResourceContent`, `Mushaf`, `Recitation`, `Morphology::Word` |

**Concerns clés** (`app/models/concerns/`) :

| Concern | Rôle |
|---|---|
| `Resourceable` | Scope les lignes de contenu à `resource_content_id` |
| `HasMetaData` | Helpers colonne JSON `meta_data` sur `ResourceContent` |
| `PaperTrailAttribution` | Qui a changé quoi |
| `Slugable` | Slugs URL pour les ressources téléchargeables |
| `NavigationSearchable` | Indexation recherche admin/globale |

**Namespaces à connaître :**

- `app/models/draft/` — Brouillons CMS avant publication
- `app/models/morphology/` — Graphe linguistique (phrases, grammaire, dépendances)
- `app/models/audio/` — Sous-modèles récitation audio
- `app/models/segments/` — Modèles outils segments + connexion SQLite dynamique
- `app/models/tools/` — Classes vérification intégrité (pas contrôleurs HTTP)

### `app/presenters/` (~44 fichiers) — View models

**On sait :** QUL utilise beaucoup les presenters. Les contrôleurs définissent `@presenter` ou appellent les classes presenter.

| Modèle | Exemple |
|---|---|
| Presenters de page | `ResourcesPresenter`, `LandingPresenter` |
| Mise en forme ressource | `TranslationPresenter`, `TafsirPresenter`, `MushafLayoutResourcesPresenter` |
| API v1 | `app/presenters/v1/chapter_presenter.rb`, `verse_presenter.rb` |

**On pense :** Si une vue a besoin de données calculées ou d'URLs CDN, cherche un presenter avant l'ERB.

### `app/services/` (~26 fichiers) — Petite orchestration

Ce n'est pas la couche métier principale — c'est `lib/`. Ici, on a du **colle à portée app** :

| Service | Rôle |
|---|---|
| `DocsPageService` / `DocsManifest` | Charger la doc markdown depuis `app/views/docs/markdown/` |
| `Resources::SearchQuery` | Recherche catalogue ressources |
| `Morphology::PhraseNodeService` | Mutations graphe de phrases |
| `Search::ArabicNormalizer` | Normalisation texte arabe pour la recherche |

**Règle simple :** import/export en masse → `lib/`. Orchestration par requête → `app/services/`.

### `app/jobs/` (~30 fichiers) — Workers Sidekiq

| Dossier / fichier | Rôle |
|---|---|
| `draft_content/` | Approuver/importer/vérifier contenu brouillon |
| `export/` | Exports tafsir, mushaf layout |
| `audio/` | Générer fichiers, découper gapless, export segments |
| `segments/` | Export segments récitateur |
| `recurring/` | Mises à jour stats API |
| `async_resource_action_job.rb` | Dispatcher générique `resource.send(action)` |
| `backup_job.rb` | Sauvegarde DB quotidienne (prod uniquement) |

**Modèle de traçage :** bouton admin → `perform_later` → job → classe `lib/` → mise à jour model/S3.

### `app/views/` (~509 fichiers) — Templates ERB

| Dossier | Contenu |
|---|---|
| `landing/` | Page d'accueil |
| `resources/` | Catalogue, aperçus par type de ressource (`previews/_*.html.erb`) |
| `docs/` | Chrome docs + fichiers **source** `markdown/*.md` |
| `community/` | Index outils, shell optimiseur SVG |
| `translation_proofreadings/`, `tafsir_proofreadings/` | UI relecture contributeur |
| `morphology_phrases/`, `mushaf_layouts/` | Outils d'édition communautaire |
| `segments/` | Tableau de bord segments (héberge les points de montage Vue) |
| `api/v1/` | Jbuilder/props JSON pour réponses API |
| `tools/header_alert/` | Bannières d'aide contextuelle par outil |
| `shared/` | Partials transversaux (affichage ayah, recherche, mushaf SVG) |

**On sait :** Les partials sous `app/views/resources/previews/` correspondent 1:1 à `DownloadableResource::RESOURCE_TYPES`.

### `app/javascript/` (~148 fichiers) — Frontend

| Chemin | Tech | Quand |
|---|---|---|
| `controllers/` (~91 contrôleurs Stimulus) | Stimulus | La plupart des outils interactifs contributeur |
| `segments/` | Vue 3 + esbuild | Constructeur de segments audio |
| `svg/` | Vue 3 | Outil optimiseur SVG |
| `active_admin/` | Helpers jQuery | JS CMS uniquement |
| `application.js` | Entrée | Turbo + enregistrement Stimulus |

**Build :** `package.json` → esbuild (`npm run build`) + Tailwind (`npm run build:css`). Rechargement dev via `Procfile.dev`.

### Dossiers `app/` plus petits

| Dossier | Rôle |
|---|---|
| `app/finders/v1/` | Objets requête API (ex. `SegmentFinder`) |
| `app/components/` | ViewComponent (minimal — `SplitScreenComponent`) |
| `app/helpers/` | Helpers de vue ; `tools_helper.rb` enregistre les outils contributeur |
| `app/mailers/` | Emails mise à jour export, fin de segment, mail utilisateur |
| `app/assets/stylesheets/` | Source SCSS/Tailwind, surcharges Active Admin |
| `app/channels/` | Action Cable (si utilisé — faible visibilité en onboarding) |

---

## `lib/` — pipelines métier (~164 fichiers)

**Important maintenant :** les contributeurs seniors passent beaucoup de temps ici pour les données.

```text
lib/
├── importer/          ← Tirer des sources externes VERS la DB Quran
├── exporter/          ← Sérialiser la DB Quran VERS des fichiers
├── audio/             ← Génération fichiers audio, découpage gapless, métadonnées
├── audio_segment/     ← Adaptateurs format segment (tarteel, ayah-by-ayah, …)
├── layout_exporter/   ← Calcul layout page mushaf
├── corpus/            ← Enums types morphologie / HTML grammaire
├── tajweed_annotation/← Tokenisation tajweed
├── utils/             ← Partagé : sanitizers, recherche, sauvegardes, helpers Quran
├── tasks/             ← Tâches Rake (28 fichiers) — migrations & imports ponctuels
├── api/               ← Helpers paramètres API
├── active_admin/      ← Inputs Active Admin personnalisés
├── data/              ← Données seed JSON statiques
└── (top-level)        ← export_service.rb, tool_card.rb, cloudflare_cache_clearer.rb, …
```

### `lib/importer/` (9 classes)

| Classe | Source |
|---|---|
| `Importer::QuranEnc` | Traductions QuranEnc |
| `Importer::QuranEncTafsir` | Tafsir QuranEnc |
| `Importer::IslamEnc` | IslamEnc |
| `Importer::QuranAcademy`, `QuranKsuEduTafsir`, `QuranTafsirNet`, `TafsirApp`, `EQuranLibrary` | Diverses sources tafsir/traduction |
| `Importer::Base` | Échafaudage import partagé |

**Point d'entrée :** `Importer::Base` + un importeur concret.

### `lib/exporter/` (26 classes)

| Classe | Exporte |
|---|---|
| `Exporter::DownloadableResources` | **Orchestrateur** — dispatch `export_all`, `refresh_export!` |
| `Exporter::ExportTranslation` | Variantes JSON/SQLite traduction |
| `Exporter::ExportTafsir` | Packages tafsir |
| `Exporter::ExportMushafLayout` | Données layout mushaf |
| `Exporter::ExportSurahRecitation` / `ExportAyahRecitation` / `ExportWordRecitation` | Manifestes audio |
| `Exporter::ExportQuranicMorphology` | Dumps morphologie |
| `Exporter::ExportFont` | Fichiers de polices |
| `Exporter::BaseExporter` | Utilitaires export partagés |

**On sait :** `Exporter::DownloadableResources#create_download_file` zippe la sortie et l'attache à `DownloadableFile` via Active Storage.

### `lib/tasks/` — tâches rake opérationnelles

| Fichier tâche | Rôle |
|---|---|
| `dump.rake` | Dumps base de données |
| `import_*.rake` | Imports sources ponctuels (qaloun, warsh, lemmas, topics) |
| `audio.rake`, `audio_segments.rake` | Maintenance pipeline audio |
| `quran_scripts.rake` | Corrections variantes de script |
| `migrate_*.rake` | Migrations de données hors migrations AR |
| `tailwindcss.rake` | Hook build CSS |

**On pense :** Beaucoup de tâches sont lancées manuellement en prod/staging. Pas en CI.

---

## `config/` — câblage (~64 fichiers)

| Fichier / dossier | Ce qu'il contrôle |
|---|---|
| `routes.rb` | **Commence ici** pour trouver les URLs |
| `database.yml` | Connexions deux DB (CMS + Quran) |
| `storage.yml` | Services Active Storage (`local`, `qul_exports`, `qul_segments`, `database_backups`) |
| `docs.yml` | Manifeste sidebar docs (catégories + slugs de pages) |
| `initializers/sidekiq.rb` | Adaptateur jobs, scheduler, Sidekiq Web |
| `sidekiq_scheduler.yml` | Cron : sauvegarde quotidienne, vérification hebdomadaire QuranEnc |
| `resources/` | `surah_name_aliases.yml` et config ressources |
| `environments/` | Memcached en prod, cache null en dev test |
| `locales/` | i18n (surface limitée) |

---

## `db/` — schéma CMS uniquement

| Chemin | Rôle |
|---|---|
| `schema.rb` | **Tables CMS uniquement** (~40 tables) — users, downloads, drafts, versions |
| `migrate/` (~105 fichiers) | Migrations CMS |
| `seeds.rb` | Données seed minimales |

**On sait :** Pas de `schema.rb` pour les tables Quran. Le schéma Quran vient du dump SQL (Phase 17).

---

## `bin/` + `docker/` + fichiers processus racine

| Fichier | Rôle |
|---|---|
| `bin/setup` | `bundle install`, `db:create:all`, `db:prepare`, `npm install` |
| `bin/dev` | Foreman → Rails + esbuild watch + Tailwind watch |
| `Procfile.dev` | Définitions de processus pour le dev |
| `docker/dev/init-db.sh` | Crée la DB `quran_dev` + schéma `quran` |
| `Dockerfile` | Image production (utilisée par le déploiement GitHub Actions) |

---

## Documentation — trois emplacements, une vérité

| Emplacement | Statut |
|---|---|
| `app/views/docs/markdown/*.md` | **On sait — source canonique** rendue à `/docs/:key` |
| `config/docs.yml` | **On sait — manifeste de navigation** (catégories, ordre) |
| `docs/` (racine dépôt) | **On sait — presque vide** (`superpowers/` uniquement) ; ignorer pour la doc utilisateur |
| `README.md` | Liens vers la doc du site live sur qul.tarteel.ai/docs |
| `.github/copilot-instructions.md` | Guide setup mainteneur/agent (instructions env détaillées) |

**Services impliqués :** `DocsManifest` lit `config/docs.yml` ; `DocsPageService` charge les fichiers markdown.

---

## `test/` — couverture légère (~18 fichiers)

| Chemin | Ce qui est testé |
|---|---|
| `test/lib/` | Une partie du code lib corpus/morphologie |
| `test/services/` | Quelques objets service |
| `test/test_helper.rb` | Setup test Rails standard |

**On pense :** Les modèles Quran sont souvent stubés ou non testés. La vérification manuelle et les diagnostics admin (`Tools::DataIntegrityChecks`) compensent.

**On ne sait pas :** La stratégie de test complète — la Phase 20 approfondira.

---

## `scripts/` — utilitaires ops (~22 fichiers)

Pas partie du runtime. Utilisés pour polices, audio, captures d'écran :

- `genrate_font_glyph.rb`, `subset_fonts.sh`, `woff2.rb` — pipeline polices
- `mp3_to_wave.sh`, `optimize_audio.sh` — préparation audio
- `cypress-e2e/` — échafaudage tests navigateur
- `screenshots.py` — captures d'écran docs

**Règle contributeur :** ne pas appeler ces scripts depuis le code app. Ce sont des helpers manuels/CI.

---

## Aide-mémoire — « J'ai besoin de trouver X »

| J'ai besoin de… | Commencer ici |
|---|---|
| Ajouter une URL publique | `config/routes.rb` → nouvelle action contrôleur → vue |
| Modifier la page de téléchargement ressource | `app/controllers/resources_controller.rb` + `app/views/resources/` |
| Corriger du JSON obsolète sur `/resources` | `DownloadableResource#refresh_export!` → `lib/exporter/downloadable_resources.rb` |
| Tracer un import | `app/admin/content/resource_content.rb` → `lib/importer/` |
| Ajouter un champ/filtre admin | `app/admin/<domain>/` |
| Comprendre les permissions | `app/models/ability.rb` + `ResourcePermission` |
| Trouver la liste des outils contributeur | `app/helpers/tools_helper.rb` |
| Éditer la doc du site | `app/views/docs/markdown/` + `config/docs.yml` |
| Ajouter un comportement Stimulus | `app/javascript/controllers/` |
| Ajouter une UI Vue | `app/javascript/segments/` ou `svg/` |
| Déboguer le travail en arrière-plan | `app/jobs/` + Sidekiq Web (`/sidekiq`) |
| Exécuter une correction de données ponctuelle | `lib/tasks/*.rake` |
| Voir la forme des tables CMS | `db/schema.rb` |
| Voir la forme des tables Quran | Charger le dump localement (Phase 17) ; grep les modèles |

---

## Trois recettes de parcours (pratique maintenant)

### Recette A — « Où vit la relecture de traduction ? »

```text
/tools  →  ToolsHelper#developer_tools (entrée ToolCard)
       →  /translation_proofreadings  (routes.rb)
       →  TranslationProofreadingsController
       →  Draft::Translation (modèle CMS)
       →  admin approve  →  DraftContent::ApproveDraftTranslationJob
       →  Translation (DB Quran)
       →  refresh_export!  →  lib/exporter/export_translation.rb
```

### Recette B — « Où est l'export layout mushaf ? »

```text
/mushaf_layouts  (outil contributeur)
/cms  →  admin/quran/mushaf*.rb
      →  Export::MushafLayoutExportJob
      →  lib/exporter/export_mushaf_layout.rb
      →  lib/layout_exporter/
      →  DownloadableFile (S3)
```

### Recette C — « Où est la liste des chapitres API ? »

```text
GET /api/v1/chapters
  →  app/controllers/api/v1/chapters_controller.rb
  →  app/presenters/v1/chapter_presenter.rb
  →  app/views/api/v1/chapters/*.json.props
  →  Chapter < QuranApiRecord
```

---

## Signaux d'échelle du dépôt

| Signal | Ce que ça veut dire pour toi |
|---|---|
| 113 fichiers admin | Le CMS est mature ; apprends Active Admin tôt |
| 91 contrôleurs Stimulus | Les outils contributeur sont JS-heavy mais pas React-first |
| 26 exporteurs vs 9 importeurs | Surface export > surface import ; les exports sont le contrat public |
| 105 migrations CMS | Le schéma évolue ; les données Quran ne migrent pas via Rails |
| 2 points d'entrée Vue seulement | Pas de grosse bibliothèque de composants — c'est ERB-first |

---

## Incertitudes

| # | Question | Label |
|---|---|---|
| 1 | Liste complète des tables DB Quran | **On ne sait pas** jusqu'au chargement du dump (Phase 17) |
| 2 | Si `docs/superpowers/` est utilisé en production | **On ne sait pas** — seulement 1 fichier à la racine `docs/` |
| 3 | Couverture Cypress e2e complète | **On ne sait pas** — scripts existants, portée floue |
| 4 | Quelles tâches rake sont encore exécutées en prod vs historiques | **On pense** — beaucoup de `one_time`/`migrate_*` semblent legacy |

---

## Résumé Phase 7

Le dépôt se divise clairement :

- **`app/`** = HTTP, CMS, vues, orchestration légère, jobs
- **`lib/`** = pipelines import/export/audio/corpus (le vrai moteur métier)
- **`config/` + `db/`** = câblage et schéma CMS
- **`app/views/docs/markdown/`** = source docs (pas `docs/`)

Quand tu es perdu : **`routes.rb` → contrôleur → presenter/model → `lib/` si les données bougent**.

---

## Arrêt ici — questions avant la Phase 8

La Phase 8 parcourra **un flux CMS** (`ResourceContent` via Active Admin : brouillon → approbation → export).

Avant cela :

1. Choisis un outil depuis `/tools` — peux-tu le tracer avec les Recettes A/B/C ?
2. Quel dossier vérifier pour un bug SQLite exporté vs un bug dans le formulaire de relecture ?
3. Y a-t-il un dossier qui reste une boîte noire pour toi ?

Réponds avec tes questions, ou dis **« proceed »** pour la **Phase 8 — Flux Active Admin / CMS**.

---

*Version A2 — français simple. Généré pendant l'onboarding contributeur QUL.*

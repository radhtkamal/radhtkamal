# Phase 20 — Tests et intégrité des données

> **Série d'intégration :** plongée progressive pour devenir un contributeur légitime de QUL.  
> **Prérequis :** [Phases 1–19](phase-01-what-is-qul.md)  
> **Ce fichier :** comment QUL vérifie la qualité du code et des données — tests automatisés, linters, outils d'intégrité CMS, et ce qui n'est *pas* bloqué en CI.

---

## Le titre honnête

Le contrôle qualité de QUL est **superposé mais inégal** :

| Couche | Existe ? | Bloque le merge en CI ? |
|---|---|---|
| Tests unitaires Minitest (`test/`) | Oui — **~17 fichiers**, surtout services | **Non** — pas de workflow test dans `.github/workflows/` |
| RuboCop (style Ruby) | Oui | **Non** — pas dans les workflows CI observés |
| ESLint (JS/Vue) | Oui (`package.json`) | **Non** |
| Scan sécurité CodeQL | Oui | Oui (sur les PR `main`) |
| Vérifications d'intégrité des données CMS | Oui — ~25 diagnostics admin | Manuel — le mainteneur exécute dans `/cms` |
| Hooks import/post-approve | Oui — rapporte les problèmes, bloque rarement | Pas de blocage automatique |
| Tests manuels contributeur | Attendu | Le template PR le demande |

**INFÉRENCE :** Pour les contributeurs OSS, **votre description de PR et votre vérification locale comptent plus qu'un badge CI vert**. C'est typique des apps Rails orientées données avec une couverture automatisée fine.

---

## Tests automatisés (Minitest)

### Framework et emplacement

| Élément | Détail |
|---|---|
| Framework | **Minitest** (défaut Rails) — pas de répertoire `spec/` |
| Racine des tests | `test/` |
| Helper | `test/test_helper.rb` |
| Tout exécuter | `bin/rails test` |
| Un fichier | `bin/rails test test/services/audio/segment_validator_test.rb` |

**FAIT** — Il n'y a **pas de bloc `group :test`** dans le `Gemfile` ; Minitest est fourni via Rails/Bundler (`minitest` dans `Gemfile.lock`).

### Ce qui est réellement testé

Les fichiers de test actuels se regroupent autour de la **logique pure** qui n'a pas besoin d'une base Quran complète :

| Domaine | Exemple de fichier de test |
|---|---|
| Recherche de ressources | `test/services/resources/search_query_test.rb` |
| Parsing de références Quran | `test/services/resources/quran_reference_parser_test.rb` |
| Recherche/normalisation arabe | `test/services/search/arabic_normalizer_test.rb` |
| Validation de segments audio | `test/services/audio/segment_validator_test.rb` |
| Treebank morphologie | `test/services/morphology/treebank/*_test.rb` |
| Tags POS corpus | `test/lib/corpus/pos_tags_locale_test.rb` |

### Style de test : modèles stubbés, pas Rails complet

**FAIT** — `test/test_helper.rb` définit des constantes **factices** `Chapter`, `Verse` et `ResourceContent` quand elles manquent — les tests s'exécutent sans charger `quran_dev`.

```ruby
# test/test_helper.rb pattern
Chapter.records = [FakeChapterRecord.new(...)]
Verse.records = [FakeVerseRecord.new(verse_key: '2:255', ...)]
```

**INFÉRENCE :** C'est intentionnel — les tests sont rapides et ne nécessitent pas le mini dump. Cela signifie aussi que les **chemins d'intégration** (contrôleurs, Active Record sur deux DB, jobs d'export) sont largement **non testés** en CI.

### Quand ajouter un test

| Type de changement | Attente de test |
|---|---|
| Nouveau/changé service dans `app/services/` | **Fortement encouragé** — suivez les fichiers voisins dans `test/services/` |
| Logique pure lib (`lib/corpus/`, parsers) | Ajoutez `test/lib/...` |
| Glue contrôleur uniquement | Test navigateur manuel souvent accepté |
| Câblage bouton CMS admin | Manuel |
| Migration de données / contenu Quran | Vérifications d'intégrité + spot verify manuel |

**Exemple :** Si vous corrigez `Audio::SegmentValidator`, étendez `test/services/audio/segment_validator_test.rb` avec un cas de régression (le fichier documente déjà un bug de récitation réel).

---

## Linters et formateurs

### Ruby — RuboCop

```bash
bundle exec rubocop
bundle exec rubocop -a path/to/file.rb   # auto-correct safe cops
```

**FAIT** — `.rubocop.yml` cible Ruby 3.3, Rails 8.0 ; inclut `rubocop-rails`, `rubocop-performance`, `rubocop-minitest`.

Exclusions : `db/schema.rb`, `db/migrate/**`, `bin/**`, `config/environments/*`.

### JavaScript / Vue — ESLint

```bash
yarn lint
yarn lint:fix
```

**FAIT** — `package.json` `lint-staged` exécute Prettier + ESLint sur `app/**/*.{js,jsx}` et lint ERB sur les vues.

### Lint ERB

`lint-staged` référence `bundle exec erblint` pour `app/views/**/*.html.erb`, mais **erblint n'est pas dans le `Gemfile`** selon cette enquête.

**INFÉRENCE** — Le lint ERB pre-commit peut échouer en local sauf si erblint est installé séparément. Pas une barrière CI.

---

## Workflows CI (ce qui s'exécute sur GitHub)

| Workflow | Fichier | Objectif |
|---|---|---|
| CodeQL | `.github/workflows/codeql.yml` | Analyse sécurité sur push/PR `main` |
| Deploy | `.github/workflows/deploy.yml` | Déploiement production (pas une barrière test pour contributeurs) |

**FAIT** — Pas de job `rails test` ou `rubocop` dans `.github/workflows/`.

**INFÉRENCE** — Les mainteneurs s'appuient sur la revue + vérification manuelle. Ne supposez pas que la CI détectera un test cassé que vous n'avez pas exécuté en local.

---

## Tests E2E (Cypress)

**FAIT** — Cypress vit dans un **package séparé** : `scripts/cypress-e2e/`

| Spec | Couvre |
|---|---|
| `signinTests.cy.js` | Connexion Devise |
| `signupTests.cy.js` | Inscription |

```bash
cd scripts/cypress-e2e
npx cypress open    # interactive
npx cypress run     # headless
```

**INFÉRENCE** — La couverture E2E est minimale (auth uniquement). Outils contributeur, CMS, exports ne sont **pas** couverts par Cypress dans ce dépôt.

Le `package.json` racine liste `cypress` en devDependency mais les tests s'exécutent depuis le sous-dossier.

---

## Intégrité des données — le vrai système QA Quran

Pour la qualité des **données** coraniques, QUL investit dans les **diagnostics admin** plus que dans les tests automatisés.

### Point d'entrée tableau de bord CMS

**URL :** http://localhost:3000/cms (dashboard)

**FAIT** — Le panneau **"Data Integrity checks"** liste toutes les vérifications de `Tools::DataIntegrityChecks.checks` avec des liens vers :

```
/cms/data_integrity_check?check_name=<check_name>
```

Implémentation :

| Pièce | Chemin |
|---|---|
| Définitions des vérifications | `app/models/tools/data_integrity_checks.rb` (~1200 lignes) |
| Page admin | `app/admin/tools/data_integrity_check.rb` |
| Tajweed spécifique | `app/models/tools/tajweed_rules_check.rb` |

**Note :** Le panneau Tajweed sur le dashboard est enveloppé dans `if false` — les vérifications fonctionnent toujours via URL directe si enregistrées.

### Catalogue des vérifications (représentatif)

| `check_name` | Ce qu'il trouve |
|---|---|
| `words_without_root` | Lacunes morphologiques |
| `words_without_lemma` | Attribution de lemme manquante |
| `words_without_stem` | Attribution de radical manquante |
| `words_with_missing_arabic_text` | Colonnes script vides |
| `ayah_with_missing_translations` | Trous de couverture traduction |
| `words_with_missing_translations` | Lacunes traduction mot |
| `ayah_with_missing_tafsirs` | Trous de couverture tafsir |
| `compare_translations` | Écarts entre éditions |
| `duplicate_mushaf_words` | Lignes de disposition dupliquées |
| `mushaf_words_with_incorrect_position` | Erreurs position ligne/page |
| `ayah_with_different_mushaf_page` | Même ayah sur pages différentes selon mushafs |
| `compare_two_mushaf_words` | Diff mushaf au niveau mot |
| `ayah_without_matching_ayahs` | Lacunes références ayahs similaires |

Chaque vérification renvoie un **tableau paginé** avec champs de filtre optionnels (sélecteurs mushaf, ids ressource) et liens profonds vers les pages preview CMS.

**FAIT** — Ce sont des **diagnostics en lecture seule**. Ils ne corrigent pas automatiquement les données ni ne bloquent les exports.

### Structure des vérifications

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

**INFÉRENCE** — Ajouter une nouvelle vérification d'intégrité = nouvelle méthode de classe + entrée dans le tableau `.checks`. Pas de migration nécessaire.

---

## Hooks import et approve (validation douce)

Quand du contenu est importé ou approuvé, QUL exécute des **post-hooks** qui collectent les problèmes :

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

**FAIT** — Les problèmes sont **rapportés comme commentaires CMS**, pas levés comme exceptions. L'approbation peut "réussir" avec des lacunes de données.

**INFÉRENCE** — Les mainteneurs examinent les commentaires avant `refresh_export!`.

---

## Validateurs spécifiques au domaine

### Segments audio

| Composant | Rôle |
|---|---|
| `Audio::SegmentValidator` | `app/services/audio/segment_validator.rb` |
| Tests unitaires | `test/services/audio/segment_validator_test.rb` |
| UI contributeur | `POST .../validate_segments` sur l'outil audio sourate |
| CMS | Action "Validate segments" sur les pages admin récitation |

Valide chevauchements, trous, écarts de nombre de mots, bornes de durée — catégories renvoyées comme problèmes structurés.

### Mushaf / traduction

Pas de classes validateur dédiées avec suites de tests. La qualité repose sur les vérifications SQL d'intégrité + outils de relecture humaine.

---

## Playbook de vérification manuelle (pour les PR)

Utilisez cette liste selon ce que vous avez modifié :

### Code uniquement (pas de données Quran)

```text
□ bin/rails test (ou fichier de test ciblé)
□ bundle exec rubocop sur les fichiers Ruby modifiés
□ yarn lint si JS/Vue modifié
□ smoke bin/dev : charger la page affectée dans le navigateur
```

### Données traduction / tafsir

```text
□ Expérience Phase 19 : ligne draft créée, publié inchangé
□ /cms/draft_translations filtrer par resource_content_id
□ Après approve (si applicable) : problèmes check_for_missing_translation dans commentaires CMS
□ Spot-check /ayah/:key dans le navigateur
□ Optionnel : vérification d'intégrité ayah_with_missing_translations
```

### Disposition mushaf

```text
□ Sauvegarder une page en local, vérifier les lignes MushafWord en console
□ Vérification duplicate_mushaf_words
□ Vérification mushaf_words_with_incorrect_position
□ Preview export si vous avez touché l'exporteur
```

### Segments audio

```text
□ validate_segments dans l'UI contributeur
□ bin/rails test test/services/audio/segment_validator_test.rb
□ Écoute spot d'un ayah dans le segment builder
```

### Export / downloader

```text
□ refresh_export! dans le CMS (nécessite Sidekiq + S3 ou attendez un échec)
□ Télécharger JSON/SQLite en local, vérifier que les clés verse_key joignent à votre app
□ Vérifier taille fichier et nombre de lignes vs stats vérification d'intégrité
```

---

## Attentes pour les PR

**FAIT** — `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md` inclut :

> **How Has This Been Tested?** — describe environment, commands, scenarios.

**FAIT** — `best-practices.md` recommande des PR ciblées et des scripts de vérification pour les jointures cross-resource.

**INFÉRENCE** — Écrivez des étapes manuelles explicites que les reviewers peuvent rejouer. "Tested locally" sans commandes est faible pour ce dépôt.

Bon exemple de section test PR :

```markdown
## How Has This Been Tested?
- `bin/rails test test/services/audio/segment_validator_test.rb` — pass
- `bundle exec rubocop app/services/audio/segment_validator.rb` — pass
- Loaded `/surah_audio_files/1/segment_builder?recitation_id=7`, ran Validate — 0 errors
- Ran CMS check `ayah_with_missing_translations` for resource 131 — no new gaps
```

---

## Pyramide de tests (réalité QUL)

```text
                    ┌─────────────────┐
                    │ CMS manuel +    │  ← principal pour les données
                    │ vérifications   │
                    │ d'intégrité     │
                    └────────┬────────┘
              ┌──────────────┴──────────────┐
              │  Minitest (services/lib)   │  ← en croissance, isolé
              └──────────────┬──────────────┘
        ┌────────────────────┴────────────────────┐
        │  Cypress (auth uniquement, optionnel)  │
        └────────────────────┬────────────────────┘
  ┌──────────────────────────┴──────────────────────────┐
  │  Scan sécurité CodeQL (CI)                         │
  └───────────────────────────────────────────────────┘
```

---

## Ce qui N'EST PAS testé (lacunes à connaître)

| Lacune | Risque | Atténuation |
|---|---|---|
| Pas de tests contrôleur/requête | Bugs routing/paramètres | Test URL manuel (Phase 18) |
| Pas de tests d'intégration jobs | Échecs approve/export | Exécuter Sidekiq en local, surveiller les logs |
| Pas de tests transaction deux-DB | Incohérence cross-DB | Vérification console |
| Pas de tests golden-file export | Régressions de format | Télécharger et diff l'export |
| Modèles Quran non testés en CI | Surprises callbacks AR | Vérifications d'intégrité + requêtes console spot |
| Routes `segment_pipeline` | 404 / code manquant | Éviter jusqu'à l'arrivée du contrôleur |

**Ne supposez pas** une haute couverture parce que l'app est de qualité production. La qualité production vient de la **discipline opérationnelle** (mainteneurs + outillage d'intégrité + dumps curés).

---

## Première contribution test suggérée

Si vous voulez une contribution OSS code à faible risque qui correspond au style du dépôt :

1. Trouvez un bug ou cas limite dans un service existant qui a déjà des tests (search, segment validator, treebank).
2. Ajoutez un cas Minitest qui le reproduit.
3. Corrigez le service.
4. `bin/rails test <that file>` dans la description PR.

Cela correspond mieux aux conventions existantes qu'ajouter le premier spec contrôleur du projet.

---

## Incertitudes

| # | Question | Label |
|---|---|---|
| 1 | Si les mainteneurs exécutent `bin/rails test` complet avant merge | **INFÉRENCE** — oui informellement ; pas imposé par CI |
| 2 | Si erblint est requis en local | **INCONNU** — dans lint-staged, pas dans Gemfile |
| 3 | Workflow CI test prévu | **INCONNU** — aucun aujourd'hui |
| 4 | Nombre total de vérifications d'intégrité (tajweed + data) | **FAIT** — 26 dans `DataIntegrityChecks.checks` + 4 dans `TajweedRulesCheck` |

---

## Résumé de la Phase 20

```text
bin/rails test     → tests service/unitaire rapides (modèles Quran stubbés)
bundle exec rubocop → style Ruby (local)
/cms dashboard     → vérifications d'intégrité des données (requêtes Quran DB réelles)
hooks approve      → validation douce → commentaires Admin
CI                 → CodeQL uniquement ; vous êtes responsable de la vérification manuelle dans les PR
```

Pour QUL, **l'outillage d'intégrité des données dans le CMS est aussi important que le dossier test**. Apprenez les deux avant d'expédier des changements de contenu ou d'export.

---

## Arrêtez-vous ici — questions avant la Phase 21

La Phase 21 couvre **la culture d'ingénierie et les attentes pour les PR** — comment les reviewers Tarteel/QUL pensent risque, périmètre et confiance contributeur.

1. La CI exécute-t-elle `bin/rails test` sur vos PR ?
2. Où exécutez-vous `words_without_root` en local ?
3. Que se passe-t-il quand `check_for_missing_translation` trouve des lacunes pendant l'approve — le job échoue-t-il ?

Répondez avec des questions, ou dites **"proceed"** pour la **Phase 21 — Culture d'ingénierie et attentes pour les PR**.

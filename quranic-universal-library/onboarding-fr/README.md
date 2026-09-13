# Intégration QUL — Série en français

> **Version française** de la série d'intégration contributeur Quranic Universal Library (QUL).  
> Les originaux en anglais se trouvent dans [`../onboarding/`](../onboarding/).

---

## À propos de cette série

Cette série de 23 phases est une **plongée progressive** conçue pour transformer un développeur curieux en **contributeur légitime** de QUL. Elle part du produit et du modèle de données coranique, traverse l'architecture Rails (deux bases de données, CMS, pipelines import/export), et se termine par des exercices pratiques, les normes de qualité, la culture d'ingénierie et une carte des surfaces où vous pouvez réellement aider.

Chaque phase est autonome mais s'appuie sur les précédentes. Commencez par la [Phase 1](phase-01-what-is-qul.md) si vous découvrez QUL, ou sautez à la phase qui correspond à votre besoin immédiat.

**Note :** Les 23 phases sont traduites en français dans ce dossier. Les originaux en anglais restent dans [`../onboarding/`](../onboarding/).

---

## Table des matières

| # | Titre | Description | Fichier |
|---|---|---|---|
| 1 | Qu'est-ce que la Quranic Universal Library ? | Identité produit et modèle de données — pas encore d'architecture Rails. | [phase-01-what-is-qul.md](phase-01-what-is-qul.md) |
| 2 | Le modèle de données coranique en premier | Hiérarchie, identifiants et rattachement des ressources — les données avant Rails. | [phase-02-quranic-data-model.md](phase-02-quranic-data-model.md) |
| 3 | Données sacrées, provenance et intégrité | Comment QUL garde les ressources coraniques attribuables, joignables et cohérentes. | [phase-03-provenance-and-integrity.md](phase-03-provenance-and-integrity.md) |
| 4 | Rails pour un ingénieur React/Node | Vocabulaire Rails minimum pour lire QUL — uniquement ce qui existe dans ce dépôt. | [phase-04-rails-for-react-node-engineers.md](phase-04-rails-for-react-node-engineers.md) |
| 5 | L'architecture à deux bases de données | Comment QUL sépare l'état CMS du contenu Quran, et les contraintes qui en découlent. | [phase-05-two-database-architecture.md](phase-05-two-database-architecture.md) |
| 6 | Modèle mental de l'architecture | Comment le système s'assemble — couches, frontières et chemins des données de l'édition au téléchargement. | [phase-06-architecture-mental-model.md](phase-06-architecture-mental-model.md) |
| 7 | Tour du dépôt | Où tout se trouve dans le dépôt, mappé aux couches d'architecture de la Phase 6. | [phase-07-repository-tour.md](phase-07-repository-tour.md) |
| 8 | Flux Active Admin / CMS | Parcourir une ressource représentative dans le CMS — du draft à la ligne Quran publiée au téléchargement public. | [phase-08-cms-flow.md](phase-08-cms-flow.md) |
| 9 | Flux d'exécution principal (édition) | Ce qui se passe à l'exécution quand un contributeur édite une ressource — du clic navigateur à l'écriture en base. | [phase-09-core-runtime-flow.md](phase-09-core-runtime-flow.md) |
| 10 | Pipeline d'import | Comment les données Quran externes entrent dans QUL — sources, correspondance, drafts et ce que l'import ne fait pas volontairement. | [phase-10-import-pipeline.md](phase-10-import-pipeline.md) |
| 11 | Pipeline d'export | Comment le contenu Quran publié devient les fichiers JSON/SQLite téléchargés depuis `/resources`. | [phase-11-export-pipeline.md](phase-11-export-pipeline.md) |
| 12 | Morphologie et données linguistiques arabes | Racines, lemmes, radicaux, tags grammaticaux, graphes de dépendance — et la différence avec les outils mutashabihat. | [phase-12-morphology-arabic-linguistic-data.md](phase-12-morphology-arabic-linguistic-data.md) |
| 13 | Dispositions mushaf | Structure des pages mushaf imprimées — mapping page, alignement des lignes, placement des mots et export. | [phase-13-mushaf-layouts.md](phase-13-mushaf-layouts.md) |
| 14 | Sous-système audio | Stockage, segmentation, export et diffusion des récitations coraniques. | [phase-14-audio-subsystem.md](phase-14-audio-subsystem.md) |
| 15 | Jobs en arrière-plan (Sidekiq) | Travail long hors thread requête — configuration Sidekiq, familles de jobs, patterns sync/async et comportement en échec. | [phase-15-background-jobs-sidekiq.md](phase-15-background-jobs-sidekiq.md) |
| 16 | Réalité du frontend | Ce qu'est réellement le frontend QUL — pas une SPA React, mais une UI Rails-first avec Stimulus, Turbo, jQuery et deux îlots Vue. | [phase-16-frontend-reality.md](phase-16-frontend-reality.md) |
| 17 | Configuration locale | Faire tourner QUL sur votre machine, charger les données Quran et vérifier que la stack fonctionne. | [phase-17-local-setup.md](phase-17-local-setup.md) |
| 18 | Corrélation navigateur ↔ code | Tracer ce que vous voyez dans le navigateur vers routes, contrôleurs, vues, presenters et modèles. | [phase-18-browser-code-correlation.md](phase-18-browser-code-correlation.md) |
| 19 | Une expérience d'apprentissage contrôlée | Exercice pratique reliant navigateur → contrôleur → deux bases de données → CMS, avec critères de succès et rollback. | [phase-19-controlled-learning-experiment.md](phase-19-controlled-learning-experiment.md) |
| 20 | Tests et intégrité des données | Comment QUL vérifie la qualité code et données — tests automatisés, linters, outils d'intégrité CMS et ce qui n'est pas bloqué en CI. | [phase-20-testing-and-data-integrity.md](phase-20-testing-and-data-integrity.md) |
| 21 | Culture d'ingénierie et attentes PR | Comment QUL/Tarteel pense les contributions — niveaux de risque, normes de revue, étiquette issues et PR faciles à merger. | [phase-21-engineering-culture-pr-expectations.md](phase-21-engineering-culture-pr-expectations.md) |
| 22 | Workflow git OSS | Fork → branche → sync → mécaniques PR pour `TarteelAI/quranic-universal-library` — commandes git pratiques et hygiène. | [phase-22-oss-git-workflow.md](phase-22-oss-git-workflow.md) |
| 23 | Surfaces de contribution et préparation | Où vous pouvez réellement aider selon vos compétences — plus une liste de contrôle honnête de préparation. | [phase-23-contribution-surface-readiness.md](phase-23-contribution-surface-readiness.md) |

---

## Parcours suggéré

```text
Fondations (1–3)     → Produit, données, provenance
Rails & architecture (4–6) → Vocabulaire, deux DB, modèle mental
Exploration (7–18)   → Dépôt, CMS, pipelines, domaines, setup, URLs
Pratique & contribution (19–23) → Expérience, tests, culture, git, carte
```

---

## Versions linguistiques

| Dossier | Langue | Phases disponibles |
|---|---|---|
| [`../onboarding/`](../onboarding/) | Anglais (original) | 1–23 |
| `onboarding-fr/` (ce dossier) | Français | 1–23 |

---

*Généré pendant l'intégration contributeur QUL. Version française complète (phases 1–23). Investigation en lecture seule — aucun code modifié.*

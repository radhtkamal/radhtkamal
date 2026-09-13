# Intégration QUL — Français simple (niveau A2)

> **Version A2** de la série d'intégration contributeur Quranic Universal Library (QUL).  
> On utilise un **français simple** (niveau élémentaire CEFR A2). Le contenu technique reste le même.

---

## Autres versions

| Dossier | Langue | Niveau |
|---|---|---|
| [`../onboarding/`](../onboarding/) | Anglais | Original |
| [`../onboarding-fr/`](../onboarding-fr/) | Français | Version normale |
| `onboarding-fr-a2/` (ce dossier) | Français | **A2 — français simple** |

---

## Qu'est-ce que A2 ?

A2 signifie **français élémentaire** (CEFR). Dans cette version :

- On utilise des **phrases courtes** (8–15 mots)
- On utilise des **mots simples**
- On explique les termes techniques une fois
- On garde **le même contenu technique** que la version normale
- On garde **inchangés** : code, chemins, URLs, commandes

Si tu préfères un français plus riche, utilise [`../onboarding-fr/`](../onboarding-fr/).

---

## À propos de cette série

Cette série a **23 phases**. On apprend QUL étape par étape. On part du produit et des données coraniques. On traverse l'architecture Rails (deux bases, CMS, import/export). On finit par des exercices pratiques, les tests, la culture d'ingénierie et une carte des surfaces de contribution.

Chaque phase est autonome. Elle s'appuie sur les précédentes. Commence par la [Phase 1](phase-01-what-is-qul.md) si tu découvres QUL.

**Note :** Les 23 phases sont disponibles en version A2 dans ce dossier. La version française normale est dans [`../onboarding-fr/`](../onboarding-fr/).

---

## Table des matières

| # | Titre simple | En une phrase | Fichier |
|---|---|---|---|
| 1 | Qu'est-ce que QUL ? | On découvre le produit et les données coraniques. | [phase-01-what-is-qul.md](phase-01-what-is-qul.md) |
| 2 | Le modèle de données | On apprend les identifiants et les ressources. | [phase-02-quranic-data-model.md](phase-02-quranic-data-model.md) |
| 3 | Provenance et intégrité | On voit comment QUL garde les données fiables. | [phase-03-provenance-and-integrity.md](phase-03-provenance-and-integrity.md) |
| 4 | Rails pour dev React/Node | On apprend le vocabulaire Rails minimum pour lire QUL. | [phase-04-rails-for-react-node-engineers.md](phase-04-rails-for-react-node-engineers.md) |
| 5 | Deux bases de données | On comprend la séparation CMS et contenu Quran. | [phase-05-two-database-architecture.md](phase-05-two-database-architecture.md) |
| 6 | Modèle mental | On voit comment le système s'assemble de bout en bout. | [phase-06-architecture-mental-model.md](phase-06-architecture-mental-model.md) |
| 7 | Tour du dépôt | On trouve où tout se situe dans le code. | [phase-07-repository-tour.md](phase-07-repository-tour.md) |
| 8 | Flux CMS | On suit une ressource du draft au téléchargement public. | [phase-08-cms-flow.md](phase-08-cms-flow.md) |
| 9 | Flux d'édition | On trace un clic contributeur jusqu'à la base de données. | [phase-09-core-runtime-flow.md](phase-09-core-runtime-flow.md) |
| 10 | Pipeline d'import | On voit comment les données externes entrent dans QUL. | [phase-10-import-pipeline.md](phase-10-import-pipeline.md) |
| 11 | Pipeline d'export | On voit comment le contenu devient des fichiers sur `/resources`. | [phase-11-export-pipeline.md](phase-11-export-pipeline.md) |
| 12 | Morphologie arabe | On explore racines, lemmes, radicaux et graphes. | [phase-12-morphology-arabic-linguistic-data.md](phase-12-morphology-arabic-linguistic-data.md) |
| 13 | Dispositions mushaf | On comprend les pages imprimées et le placement des mots. | [phase-13-mushaf-layouts.md](phase-13-mushaf-layouts.md) |
| 14 | Sous-système audio | On explore stockage, segmentation et export audio. | [phase-14-audio-subsystem.md](phase-14-audio-subsystem.md) |
| 15 | Jobs Sidekiq | On apprend le travail en arrière-plan hors requête web. | [phase-15-background-jobs-sidekiq.md](phase-15-background-jobs-sidekiq.md) |
| 16 | Réalité du frontend | On découvre Stimulus, Turbo, jQuery et deux îlots Vue. | [phase-16-frontend-reality.md](phase-16-frontend-reality.md) |
| 17 | Setup local | On fait tourner QUL sur sa machine avec les données. | [phase-17-local-setup.md](phase-17-local-setup.md) |
| 18 | Navigateur ↔ code | On relie ce qu'on voit dans le navigateur au code Rails. | [phase-18-browser-code-correlation.md](phase-18-browser-code-correlation.md) |
| 19 | Expérience pratique | On fait un exercice draft-first avec rollback. | [phase-19-controlled-learning-experiment.md](phase-19-controlled-learning-experiment.md) |
| 20 | Tests et intégrité | On apprend comment QUL vérifie code et données. | [phase-20-testing-and-data-integrity.md](phase-20-testing-and-data-integrity.md) |
| 21 | Culture et PR | On comprend les attentes des reviewers et les niveaux de risque. | [phase-21-engineering-culture-pr-expectations.md](phase-21-engineering-culture-pr-expectations.md) |
| 22 | Workflow git OSS | On apprend fork, branche, sync et ouverture de PR. | [phase-22-oss-git-workflow.md](phase-22-oss-git-workflow.md) |
| 23 | Où contribuer | On trouve sa surface de contribution et on s'auto-évalue. | [phase-23-contribution-surface-readiness.md](phase-23-contribution-surface-readiness.md) |

---

## Parcours suggéré

```text
Fondations (1–3)        → Produit, données, provenance
Rails & architecture (4–6) → Vocabulaire, deux DB, modèle mental
Exploration (7–18)      → Dépôt, CMS, pipelines, domaines, setup, URLs
Pratique & contribution (19–23) → Expérience, tests, culture, git, carte
```

**Conseil :** Si tu es nouveau, lis les phases 1 à 18 dans l'ordre. Ensuite, fais l'expérience de la Phase 19 en local. Les phases 20 à 23 préparent ta première contribution.

---

## Labels utilisés dans les phases A2

| Label | Signification |
|---|---|
| **On sait :** | Information confirmée dans le code ou la doc |
| **On pense :** | Déduction raisonnable, pas toujours écrite noir sur blanc |
| **On ne sait pas :** | Information non confirmée |
| **Important maintenant :** | Point critique à retenir tout de suite |

---

*Version A2 — français simple. Généré pendant l'onboarding contributeur QUL.*

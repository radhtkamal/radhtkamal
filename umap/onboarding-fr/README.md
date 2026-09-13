# Intégration uMap (français)

Parcours d'intégration en **13 phases** pour ingénieurs web expérimentés qui souhaitent comprendre uMap et contribuer à l'open source.

> Version anglaise : [`onboarding/`](../onboarding/)

---

## Parcours

| Phase | Document | Objectif |
|------:|----------|----------|
| 1 | [Qu'est-ce que uMap ?](phase-1-what-is-umap.md) | Périmètre produit, parcours utilisateur, modèle mental |
| 2 | [Primer domaine](phase-2-domain-primer.md) | GeoJSON, calques, tuiles, Overpass — uniquement ce que le dépôt utilise |
| 3 | [Architecture](phase-3-architecture.md) | Frontières Django / Leaflet / stockage |
| 4 | [Tour du dépôt](phase-4-repository-tour.md) | Où se trouve quoi dans le code |
| 5 | [Parcours runtime](phase-5-runtime-walkthroughs.md) | Tracer ouvrir → éditer → sauvegarder → couche distante |
| 6 | [Algorithmes et transformations](phase-6-algorithms-and-data-transformations.md) | Merge, import, règles, choroplèthe |
| 7 | [Exécuter en local](phase-7-run-locally.md) | PostGIS, `uv`, `local.py`, signaux de succès |
| 8 | [Relier le navigateur au code](phase-8-connect-browser-to-code.md) | DevTools, `U.MAP`, journal côté client |
| 9 | [Expérience d'apprentissage](phase-9-controlled-learning-experiment.md) | Premier changement réversible avec vérification |
| 10 | [Culture d'ingénierie](phase-10-engineering-culture.md) | Lint, i18n, AGPL, attentes des mainteneurs |
| 11 | [Workflow git OSS](phase-11-oss-git-workflow.md) | Fork → branche → PR vers `umap-project/umap` |
| 12 | [Modèle mental des tests](phase-12-testing-mental-model.md) | pytest, Playwright, Mocha |
| 13 | [Surface de contribution](phase-13-contribution-surface-and-readiness.md) | Où contribuer et évaluer sa préparation |

---

## Comment utiliser ce parcours

1. Lisez les phases **dans l'ordre** — chaque phase s'appuie sur la précédente.
2. Les phases **1 à 6** sont en lecture seule (archéologie du code).
3. La **phase 7** est le point de bascule opérationnel (installation locale).
4. Les phases **8 à 9** supposent une instance locale en marche.
5. Les phases **10 à 13** préparent une contribution réelle.

Pour avancer entre les phases, dites par exemple **« continuer vers la Phase 2 »** dans la conversation d'intégration.

---

## Libellés de preuve

Les documents utilisent des libellés pour distinguer ce qui est établi du dépôt :

| Libellé | Signification |
|---------|---------------|
| **FAITE** / **OBSERVÉ** | Directement issu du code, des tests ou de la documentation du dépôt |
| **INFÉRENCE** | Conclusion raisonnable à partir des preuves |
| **INCONNU** | Non établi à partir de ce dépôt seul |

Les phases 1 à 4 emploient surtout **FAITE** ; à partir de la phase 5, **OBSERVÉ** remplace **FAITE** (aligné sur la version anglaise).

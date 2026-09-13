# Phase 21 — Culture d'ingénierie et attentes pour les PR

> **Série d'intégration :** plongée progressive pour devenir un contributeur légitime de QUL.  
> **Prérequis :** [Phases 1–20](phase-01-what-is-qul.md)  
> **Ce fichier :** comment QUL/Tarteel pense les contributions — niveaux de risque, normes de revue, étiquette des issues, et ce qui rend une PR facile à merger.

---

## Deux chemins de contribution légitimes

QUL sépare délibérément le **code plateforme** des **données coraniques** :

| Chemin | Vous modifiez | Entrée typique | Focus de revue |
|---|---|---|---|
| **Code** | App Rails, exporteurs, UI, site docs | Fork → PR sur GitHub | Correctness, périmètre, régressions |
| **Données** | Traductions, segments audio, mushaf, morphologie | CMS + outils de relecture | Exactitude, provenance, clés de jointure |

**FAIT** — `contribute-code.md` et `contribute-data.md` sont des docs séparées sur `/docs`.

**INFÉRENCE :** Une PR qui mélange une grosse migration de données avec un refactor UI sera difficile à reviewer. Séparez-les.

---

## Valeurs culturelles (inférées des docs + comportement du dépôt)

Ce n'est pas écrit comme un manifeste, mais cela apparaît de façon cohérente :

### 1. Les données publiées sont sacrées

- Les outils contributeur écrivent des **drafts** d'abord (Phase 9).
- L'**approve** CMS promeut vers la Quran DB.
- L'**export** est une étape séparée et délibérée (Phase 11).
- Les vérifications d'intégrité existent parce que de mauvaises données partent vers des milliers d'apps via `/resources`.

**Mentalité reviewer :** « Que se passe-t-il pour les consommateurs downstream si c'est faux ? »

### 2. Les identifiants doivent rester stables

`verse_key`, `location`, `word_position`, `resource_content_id` sont des contrats de jointure (Phases 1–2).

**Changements à haut risque :** Renommer des clés dans les exports, changer la cardinalité, altérer la sémantique de `verse_index`.

### 3. Le périmètre bat l'ambition

**FAIT** — `best-practices.md` : *"Keep pull requests focused by concern (docs, integrations, feature)."*

Les PR mergées récemment sont petites et descriptives :

```text
Fix typo in quran-script group description (#745)
fix: paginate word mistakes by per_page instead of hardcoded 100 (#721)
docs: clarify that audio_url points at Tarteel's CDN (#693)
```

**INFÉRENCE :** Un changement logique par PR. Typos et corrections ciblées mergent rapidement.

### 4. La documentation est un produit

Les lacunes de setup doivent être corrigées via PR (`project-setup.md` l'invite explicitement).

**Alerte chemin obsolète :** `contributing.md` dit d'éditer `docs/` — **faux**. La doc canonique vit dans `app/views/docs/markdown/` + `config/docs.yml` (Phases 1, 18).

### 5. Coordination communautaire pour le gros travail de données

**FAIT** — `contribute-data.md` pointe vers [Discord](https://t.zip/discord?utm_source=github&utm_campaign=qul) pour coordonner les efforts de données plus importants.

**INFÉRENCE :** Ne re-segmentez pas silencieusement une récitation complète ni ne réimportez une traduction sans alignement mainteneur.

---

## Niveaux de risque (comment les reviewers classifient votre changement)

Utilisez ceci pour vous auto-trier avant d'ouvrir une PR :

| Niveau | Exemples | Barre de revue | Votre travail |
|---|---|---|---|
| **Faible** | Typo, CSS, fix doc, test pour service pur | Légère | Montrer la sortie des commandes |
| **Moyen** | Nouveau comportement Stimulus, tweak recherche, champ exporteur | Moyenne | Tests + URL manuelle |
| **Élevé** | Job approve, pipeline import, format export | Lourde | Tests, vérification d'intégrité, diff export exemple |
| **Critique** | Schéma Quran, remplacement bulk traduction, édition mass mushaf | Piloté mainteneur | Issue d'abord, PRs par phases |

**INFÉRENCE :** Le travail de niveau critique sans issue va probablement stagner.

---

## Issues avant les PR

**FAIT** — Le template PR indique :

> *"This project only accepts pull requests related to open issues"*  
> *"If suggesting a new feature or change, please discuss it in an issue first"*

**Interprétation pratique :**

| Situation | Action |
|---|---|
| Typo évidente / fix commande doc | PR ok, issue optionnelle |
| Nouvelle fonctionnalité ou changement de comportement | Ouvrir une issue d'abord |
| Correction de données dans une ressource publiée | Utiliser le template d'issue structuré |
| Vouloir accès CMS / projet | Issue candidature contributeur |

---

## Templates d'issues GitHub (culture données)

Les templates structurés encodent **ce que les mainteneurs ont besoin** :

| Template | Labels | Exigence notable |
|---|---|---|
| Issue traduction | `content issue`, `translation` | Erreurs factuelles uniquement — pas préférence de formulation |
| Script Quran / police | `content issue`, `script` | Sourate, ayah, type de script |
| Disposition mushaf | `layout issue` | Nom disposition + numéro de page |
| Nouvelle disposition mushaf | `mushaf layout` | URL PDF, nombre de pages, lignes par page |
| Candidature contributeur | `contributor-request` | Reconnaissance licence OSS bénévole |

**FAIT** — Le template traduction dit explicitement :

> *"This report is for typos, spelling mistakes, and factual issues only. Please do not use this form to suggest changes in wording preferences or subjective opinions."*

**INFÉRENCE :** QUL traite les traductions comme des **éditions faisant autorité**, pas des paraphrases réécrites par la foule. Les disputes théologiques ou de style passent par les canaux savants appropriés, pas des PR opportunistes.

**FAIT** — Plusieurs templates contenu assignent automatiquement `naveed-ahmad` — **INFÉRENCE :** mainteneur central pour le triage contenu.

---

## Obtenir l'accès contributeur (CMS / projets)

Les contributeurs code peuvent exécuter l'app en local avec super-admin `db:seed` (Phase 17).

La **contribution de données style production** nécessite :

1. **Compte GitHub** + outils de relecture (certaines éditions nécessitent une connexion)
2. Approbation **`UserProject`** pour un `ResourceContent` spécifique — `can_manage?` vérifie ceci (Phase 18)
3. Optionnel : issue **Contributor Application** (`.github/ISSUE_TEMPLATE/contributor-application.yml`) pour coordination bénévole

```yaml
# Contributor application asks:
# - How would you like to contribute?
# - Agreement: volunteer, open-source license
```

**FAIT** — La colonne `users.approved` existe sur le modèle User — **INFÉRENCE :** l'inscription peut nécessiter une approbation admin pour l'accès complet (barrière exacte pas entièrement tracée ; super_admin contourne les vérifications projet).

---

## Anatomie d'une PR (ce que les reviewers regardent)

### Titre

Patterns de l'historique récent :

| Style | Exemple |
|---|---|
| Fix impératif | `Fix typo in quran-script group description` |
| Préfixe conventionnel | `fix:`, `docs:`, `feat:` (utilisé parfois, pas obligatoire) |
| Fonctionnalité descriptive | `Add audio repeat and use Digitalkhatt script in segments` |

**Ligne directrice :** Le lecteur doit savoir **quoi** a changé sans ouvrir le diff.

### Description (utilisez le template)

```markdown
## Description
## Related Issue        ← link #123
## Motivation and Context
## How Has This Been Tested?   ← playbook Phase 20
## Screenshots (if appropriate)
```

**Faible :** "Tested locally"  
**Fort :** Commandes exécutées, URLs cliquées, nom vérification d'intégrité, extrait export avant/après

### Périmètre du diff

| Bon | À éviter |
|---|---|
| Une fonctionnalité ou fix | Refactors opportunistes dans la même PR |
| Style existant respecté | Nouvelle abstraction pour un seul appel |
| Tests quand on touche `app/services/` | Changer la forme JSON export sans note de migration |

### Captures d'écran

Attendues pour : changements UI, boutons CMS admin, rendu arabe, segment builder, preview mushaf.

---

## Thèmes de revue de code (ce qui reçoit des commentaires)

Basé sur l'architecture à travers les phases d'intégration :

### Sécurité des données

- Est-ce que ceci écrit directement dans `translations` / `Audio::Segment` quand ça devrait utiliser des drafts ?
- Est-ce que ça désactive PaperTrail pendant des écritures bulk sans restaurer ?
- Est-ce que ça suppose `verses.id` quand l'UI utilise `verse_key` ?

### Performance

- `find_each` / `insert_all` pour le bulk — pas `each` + `save` sur 6236 lignes
- Requêtes N+1 dans les pages index admin
- Export chargeant des tables entières en mémoire

### Compatibilité export

- Les clés `verse_key` / `location` sont-elles préservées dans le JSON ?
- Les downloaders existants casseront-ils si les noms de champs changent ?
- Le hotlinking CDN est-il documenté si on ajoute des exemples `audio_url` ?

### Frontend

- Stimulus vs nouvelle île Vue — Stimulus par défaut (Phase 16)
- `data-turbo="false"` quand les widgets jQuery cassent
- Classes polices arabes (`qpc-hafs`, etc.) pour preview script

### Docs

- Éditer `app/views/docs/markdown/`, mettre à jour `config/docs.yml` si ajout de pages
- Re-exécuter les commandes setup que vous avez changées
- Noter l'impact consommateur dans le corps de la PR

---

## Ce qui tend à être rejeté ou retardé

| Changement | Pourquoi |
|---|---|
| Reformulation massive de traduction non sollicitée | Savant / licence — utiliser le template d'issue |
| Rupture format export sans versioning | Les apps downstream dépendent de formes stables |
| Mise à jour mass Quran DB directe dans le chemin contributeur | Contourne draft/revue |
| PR sans issue pour nouvelle fonctionnalité | Le template demande explicitement une issue |
| Mélange de refactors sans lien | Fatigue de revue |
| Dépend de code manquant (contrôleur `segment_pipeline`) | Routes cassées |
| Hotlinking CDN production dans docs sans avertissement self-host | Politique (voir PR tutoriel récitation #693) |

---

## Licence et attribution

**FAIT** — Case à cocher candidature contributeur :

> *"I understand that this is a volunteer-based project and my contributions will be used under an open-source license."*

**INFÉRENCE :** Les contributions code et données sont OSS — consultez le fichier `LICENSE` du dépôt pour les termes exacts avant de contribuer un travail substantiel.

**FAIT** — Les ressources portent des métadonnées de provenance (`ResourceContent`, `DataSource`, champs auteur) — les PR données doivent respecter les licences sources et chaînes d'attribution (Phase 3).

---

## Canaux de communication

| Canal | Usage |
|---|---|
| **GitHub Issues** | Bugs, rapports données, discussion fonctionnalités |
| **GitHub PRs** | Changements code et docs |
| **Discord** | Coordination gros travail données, questions de processus |
| **Commentaires CMS / AdminTodo** | Workflow mainteneur interne — pas pour contributeurs externes initialement |

**INFÉRENCE :** La discussion technique publique préfère GitHub pour la traçabilité ; Discord pour la coordination temps réel.

---

## Échelle de confiance mainteneur

```text
1. Issue ou petite PR doc           → prouve communication + setup
2. Test service + PR bugfix         → prouve conventions code
3. Suggestions relecture            → prouve soin des données (drafts uniquement)
4. UserProject / candidature        → accès CMS ciblé
5. Travail pipeline export/import   → partenariat mainteneur
```

Vous n'avez pas besoin de l'étape 5 pour être un contributeur OSS **précieux**. Fixes doc, tests validateur segment, et améliorations vérifications d'intégrité sont de vraies contributions.

---

## Liste de contrôle auto-revue (avant "Create pull request")

```text
□ Un sujet par PR
□ Issue liée (si fonctionnalité/non trivial)
□ bin/rails test (fichiers pertinents) — Phase 20
□ bundle exec rubocop (Ruby modifié)
□ Étapes manuelles documentées dans le template PR
□ Pas de secrets (.env, credentials, master.key)
□ Chemin docs correct (app/views/docs/markdown/)
□ Changements données utilisent le chemin draft (pas publish direct)
□ Compatibilité export arrière considérée
□ Captures d'écran pour UI
```

---

## Exemples de descriptions PR (templates à copier)

### Fix doc

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

### Bugfix service

```markdown
## Description
Segment validator: flag ayah overlap when timestamp_to > next timestamp_from.

## Related Issue
Fixes #___

## How Has This Been Tested?
- `bin/rails test test/services/audio/segment_validator_test.rb` — 42 runs, 0 failures
- Reproduced case from recitation 1 surah 32 in segment builder validate UI
```

### Ajustement UI

```markdown
## Description
Change proofreading submit button label from "Purpose changes" to "Propose changes".

## How Has This Been Tested?
- Screenshot attached
- Submitted test draft on local translation_proofreadings — redirect + flash ok
```

---

## Incertitudes

| # | Question | Label |
|---|---|---|
| 1 | CODEOWNERS formel ou reviewers requis | **FAIT** — aucun dans le dépôt |
| 2 | Si tous les nouveaux utilisateurs ont besoin de `approved: true` avant connexion | **INCONNU** — colonne existe |
| 3 | SLA revue PR / réponse issue | **INCONNU** — projet bénévole |
| 4 | Si le chemin `docs/` de `contributing.md` sera corrigé upstream | **INCONNU** — obsolète selon cette enquête |

---

## Résumé de la Phase 21

```text
PRs code     → ciblées, testées, liées à issue si non trivial
Issues données → templates structurés, corrections factuelles, identifiants inclus
Éditions données → drafts → revue CMS → export (ne jamais sauter les couches)
Culture      → jointures stables, provenance, petites PR, docs comme produit
Accès        → UserProject / candidature contributeur pour travail CMS ciblé
```

Les reviewers protègent **les apps Quran downstream**, pas seulement l'esthétique Rails. Cadrez votre PR autour de l'impact consommateur et la revue devient collaborative.

---

## Arrêtez-vous ici — questions avant la Phase 22

La Phase 22 couvre le **workflow git OSS** — branching, rebase sur upstream, garder les forks à jour, et mécaniques d'hygiène PR.

1. Quand ouvrir une issue avant une PR ?
2. Quelle est la différence entre un chemin contribution code et contribution données ?
3. Pourquoi le template d'issue traduction rejette les rapports de « préférence de formulation » ?

Répondez avec des questions, ou dites **"proceed"** pour la **Phase 22 — Workflow git OSS**.

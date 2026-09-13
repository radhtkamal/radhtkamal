# Phase 3 — Données sacrées, provenance et intégrité

> **Série d'onboarding :** plongée progressive pour devenir un contributeur QUL légitime.  
> **Prérequis :** [Phase 1](phase-01-what-is-qul.md), [Phase 2](phase-02-quranic-data-model.md)  
> **Ce fichier :** comment QUL maintient les ressources coraniques attribuables, joignables et cohérentes en interne.

---

## La question technique

QUL gère des données coraniques et liées au Coran. La question technique n'est pas théologique — elle est pratique :

> **Comment QUL maintient-il les ressources attribuables, joignables et cohérentes en interne — et comment savons-nous qu'une modification n'a pas corrompu quelque chose en aval ?**

Cette phase sépare trois préoccupations :

1. **Identité canonique** — ce qui ne doit pas dériver (adresses ayah, positions de mots)
2. **Contenu éditable** — ce que traducteurs, relecteurs et importeurs modifient
3. **Couches de gouvernance** — drafts, approbations, versioning, permissions, contrôles d'intégrité

---

## Identité canonique vs contenu éditable

### Identité canonique (traiter comme infrastructure)

Ces champs définissent **où** le contenu vit dans la hiérarchie coranique. Les modifier est à haut risque.

| Champ | Modèle | Rôle |
|---|---|---|
| `chapter_number` / `chapter_id` | `Chapter`, `Verse`, `Word` | Quelle sourate |
| `verse_number` | `Verse` | Quelle ayah dans la sourate |
| `verse_key` | `Verse`, `Translation`, `Tafsir`, … | Adresse chaîne `"surah:ayah"` |
| `position` | `Word` | Index du mot dans l'ayah |
| `location` | `Word`, `Morphology::Word` | Adresse chaîne `"surah:ayah:word"` |
| `verse_index` | `Verse` | Séquence d'ayah globale (1–6236) |
| `word_index` / `sequence_number` | `Word` | Séquence de mots globale (unique) |

**FAIT :** Les exporteurs indexent la sortie par `verse_key` ou `location`. Les apps en aval joignent sur ceux-ci.

**INFÉRENCE :** Si vous changez `verse_key` sur une traduction de `"2:255"` à `"2:256"`, le texte se déplace silencieusement vers la mauvaise ayah dans chaque export — sans erreur runtime.

### Contenu éditable (travail éditorial normal)

Ces champs définissent **ce qui est dit** à une adresse stable :

| Champ | Modèle | Exemples |
|---|---|---|
| `text` | `Translation`, `Tafsir`, `WordTranslation` | Texte traduit / commentaire |
| Colonnes `text_*` | `Verse`, `Word` | Variantes de script arabe |
| `footnotes` / `FootNote.text` | `FootNote` | Notes savantes |
| `segments` (jsonb) | `Audio::Segment` | Données de timing mot/ayah |
| `root_id`, `lemma_id`, `stem_id` | `Word` | Assignations linguistiques |
| Champs d'analyse morphologique | `Morphology::Word`, segments, tokens | Métadonnées grammaticales |

**FAIT :** PaperTrail versionne bon nombre de ces champs de contenu à la mise à jour (voir ci-dessous).

### Métadonnées semi-stables (modifier avec prudence)

| Champ | Modèle | Notes |
|---|---|---|
| `resource_content_id` | Toutes les lignes de ressource | À quel package une ligne appartient — déplacer des lignes entre packages casse l'attribution |
| `approved` | `ResourceContent`, `Audio::Recitation`, `Morphology::Phrase` | Porte de publication |
| `start_verse_id` / `end_verse_id` | `Tafsir` | Définit la couverture de plage ayah — mauvaise plage = mauvais mapping commentaire |
| `meta_data` (jsonb) | `ResourceContent` | Clés source, text-type, flags footnote, timestamps d'import |

---

## Provenance : qui, où, quand

QUL trace la provenance au niveau du **package de ressource** (`ResourceContent`), pas habituellement par ligne ayah.

### Attribution auteur et traducteur

**FAIT :** `ResourceContent` appartient à :

```ruby
belongs_to :author, optional: true      # translators, scholars
belongs_to :language, optional: true
belongs_to :data_source, optional: true # where data was obtained
```

| Modèle | Table | Rôle |
|---|---|---|
| `Author` | `authors` (Quran DB) | Érudit/traducteur nommé ; lié à de nombreux enregistrements `ResourceContent` |
| `DataSource` | `data_sources` (Quran DB) | Origine externe (ex. QuranEnc, TafsirApp) |
| `Language` | `languages` (Quran DB) | Langue de la ressource |

**FAIT :** Les lignes `Translation` copient `language_name` et `language_id` depuis la ressource parente à l'import/approbation.

### Suivi de source dans les métadonnées

**FAIT :** `ResourceContent` stocke une provenance flexible dans `meta_data` jsonb. Exemples trouvés dans le code :

| Clé meta | Définie par | Signification |
|---|---|---|
| `source` | Importeurs | ex. `'quranenc'` |
| `quranenc-key` | Importeur QuranEnc | Identifiant source externe |
| `tafsirapp-key` | Importeur TafsirApp | Identifiant source externe |
| `draft-quranenc-import-version` | Import | Version source à l'import |
| `draft-quranenc-import-timestamp` | Import | Timestamp source |
| `quranenc-imported-version` | Post-approbation | Promu en meta canonique après approbation |
| `last-import-at` | `run_after_import_hooks` | Quand le contenu a été importé pour la dernière fois |
| `has-footnote` | Import | Si des footnotes existent |
| `text-type` | Config admin | Quelle colonne `text_*` exporter |
| `copyright` | Admin | Déclenche téléchargement restreint sur `DownloadableResource` |

**FAIT :** Après approbation draft, `run_after_import_hooks` promeut les métadonnées de version d'import draft :

```ruby
set_meta_value('quranenc-imported-version', delete_meta_value('draft-quranenc-import-version'))
set_meta_value('last-import-at', Time.zone.now.strftime(...))
```

### Copyright et permissions d'hébergement

**FAIT :** `ResourcePermission` (base CMS) trace le statut légal/hébergement par `ResourceContent` :

```ruby
enum :permission_to_host   # unknown, requested, granted, rejected
enum :permission_to_share  # unknown, requested, granted, rejected
# copyright_notice, source_info, contact_info
```

**FAIT :** `DownloadableResource#restrict_download?` vérifie `meta_value('copyright')`. Quand défini, la page ressource publique affiche `copyright_notice` au lieu des liens de téléchargement (`app/views/resources/detail.html.erb`).

**INFÉRENCE :** La provenance pour les utilisateurs en aval est en partie dans les fichiers d'export (clés, texte) et en partie sur la page détail ressource (description, copyright, change logs). Les exports eux-mêmes sont indexés par `verse_key` — le nom d'auteur n'est pas intégré dans chaque ligne JSON par défaut.

---

## Machine à états de publication

Le contenu traverse plusieurs couches de « publishedness » :

```mermaid
stateDiagram-v2
    direction LR

    [*] --> Draft: import ou suggestion
    Draft --> Review: need_review=true
    Review --> QuranContent: approve/import!
    QuranContent --> Versioned: PaperTrail on update
    QuranContent --> Exported: refresh_export!
    Exported --> PublicDownload: DownloadableResource published=true
    PublicDownload --> UserNotified: ChangeLog published + email
```

### Couche 1 : Draft (base CMS)

Les tables draft contiennent des modifications **proposées** avant qu'elles ne touchent le contenu Coran publié :

| Table | Modèle | Créé par |
|---|---|---|
| `draft_translations` | `Draft::Translation` | UI relecture, importeurs |
| `draft_tafsirs` | `Draft::Tafsir` | UI relecture, importeurs |
| `draft_word_translations` | `Draft::WordTranslation` | UI traduction de mots |
| `draft_contents` | `Draft::Content` | Pipeline draft générique |
| `draft_foot_notes` | `Draft::FootNote` | Modifications footnotes |

**FAIT :** Les lignes draft tracent l'état de relecture :

| Champ | Signification |
|---|---|
| `current_text` | Ce qui est publié maintenant |
| `draft_text` | Modification proposée |
| `text_matched` | Si le draft égale le courant (`true` = pas de vraie modification) |
| `need_review` | Nécessite relecture humaine avant import |
| `imported` | Si le draft a été appliqué au contenu Coran |
| `user_id` | Qui a suggéré la modification (flux relecture) |

**FAIT :** La relecture communautaire (`TranslationProofreadingsController`) **n'écrase pas** le texte publié directement. Elle appelle `Translation#save_suggestions`, qui crée un `Draft::Translation` avec `need_review: true`.

### Couche 2 : Approbation → contenu Coran

**FAIT :** `Draft::Translation#import!` écrit dans la table `translations` de la base Coran :

1. Trouve ou initialise `Translation` pour `verse_id` + `resource_content_id`
2. Copie `draft_text` → `text`
3. **Re-synchronise les champs d'identité** depuis le verset : `verse_key`, `chapter_id`, `verse_number`, métadonnées juz/hizb/page
4. Attribue le changement PaperTrail à l'utilisateur soumettant
5. Marque le draft `imported: true`

**FAIT :** L'import utilise `save(validate: false)` — il n'y a **pas de validations ActiveRecord** bloquant l'écriture sur les modèles de contenu centraux.

**FAIT :** L'approbation en masse s'exécute via jobs Sidekiq (`DraftContent::ApproveDraftTranslationJob`, etc.). Pendant l'import en masse, `PaperTrail.enabled = false` pour éviter des milliers de lignes de version.

### Couche 3 : Approbation ressource

**FAIT :** Le booléen `ResourceContent#approved` contrôle si une ressource est considérée approuvée. Scope : `ResourceContent.approved`.

**FAIT :** `Audio::Recitation` a son propre flag `approved`. Les récitations clonées démarrent `approved: false`.

**FAIT :** `Morphology::Phrase` a `approved` et `review_status` — seules les phrases approuvées apparaissent dans les aperçus mutashabihat publics.

### Couche 4 : Publication téléchargement public

**FAIT :** `DownloadableResource#published` contrôle la visibilité sur `/resources`. Scope : `DownloadableResource.published`.

**FAIT :** `ResourcesController#detail` n'affiche que les ressources `.published`.

**FAIT :** `refresh_export!` régénère les pièces jointes `DownloadableFile`, puis envoie éventuellement des emails aux utilisateurs ayant précédemment téléchargé la ressource.

### Couche 5 : Annonces de changement lisibles par l'humain

**FAIT :** `ChangeLog` (base CMS) est un **changelog curaté** pour les mises à jour de ressources — pas PaperTrail automatique :

```ruby
class ChangeLog < ApplicationRecord
  belongs_to :user
  belongs_to :resource_content
  validates :title, :text, :excerpt, presence: true
  scope :published, -> { where published: true }
end
```

Les mainteneurs les rédigent manuellement dans Active Admin. Quand une ressource est re-exportée, `notify_users` attache le `ChangeLog` publié le plus récent à l'email de mise à jour.

---

## Versioning : PaperTrail

**FAIT :** QUL utilise la gem [PaperTrail](https://github.com/paper-trail-gem/paper_trail). Les versions sont stockées dans la table `versions` de la base CMS :

```ruby
# db/schema.rb
create_table "versions" do |t|
  t.string "item_type"    # e.g. "Translation"
  t.integer "item_id"
  t.string "event"        # "update", "destroy"
  t.string "whodunnit"   # GlobalID of user
  t.text "object"        # YAML snapshot of previous state
  t.boolean "reviewed", default: false
  t.integer "reviewed_by_id"
  t.integer "user_id"
end
```

**FAIT :** Modèles avec PaperTrail (sélection) :

| Modèle | Événements tracés |
|---|---|
| `Verse` | update, destroy |
| `Word` | update, destroy |
| `Translation` | update |
| `Tafsir` | update |
| `WordTranslation` | update |
| `FootNote` | update |
| `Chapter` | update |
| `ChapterInfo` | update |
| `Transliteration` | update |
| `ArabicTransliteration` | update |
| `MushafWord` | update |

**FAIT :** Active Admin enregistre les versions comme **« Content Changes »** à `/cms/content_changes` avec :
- Voir version précédente/suivante
- **Revert** vers une version antérieure (`resource.reify` + `save`)
- Marquer version comme relue/non relue

**FAIT :** La concern `PaperTrailAttribution` encapsule les écritures pour définir `whodunnit` :

```ruby
PaperTrail.request(whodunnit: user.to_gid.to_s, controller_info: { user_id: user.id }) do
  yield
end
```

**INFÉRENCE :** PaperTrail trace les **modifications de champs de contenu** sur des lignes individuelles. Il ne remplace pas le workflow draft/relecture — les drafts vivent dans des tables séparées jusqu'à import explicite.

**INCONNU :** Si les versions PaperTrail pour les modèles Quran DB sont stockées dans la table CMS `versions` ou un store séparé — la table `versions` est dans `db/schema.rb` (base CMS), mais les modèles versionnés utilisent `QuranApiRecord`. Cela fonctionne probablement parce que PaperTrail utilise par défaut la connexion de base de données principale de l'application pour la table versions. Vérifier en Phase 17 en exécution locale.

---

## Contrôle d'accès : qui peut modifier quoi

**FAIT :** CanCanCan (modèle `Ability`) définit les rôles :

| Rôle | Capacités d'édition de données |
|---|---|
| `super_admin` | `can :manage, :all` |
| `admin` | Gérer traductions, tafsirs, drafts, récitations, contenu ressource |
| `moderator` | Créer/mettre à jour drafts (pas destroy) ; gérer phrases morphologie |
| `contributor` | Limité (hérite des restrictions normal_user) |
| `normal_user` | Lire la plupart des choses ; pas d'accès ressources admin |
| `audio_annotator` | Accès spécial aux ressources récitation |

**FAIT :** Les outils d'édition communautaire utilisent un **accès basé sur projet** via `UserProject` :

```ruby
# application_controller.rb
access = current_user.user_projects.find_by(resource_content_id: resource.id)
access if access&.approved?
```

Les contributeurs demandent l'accès à un `ResourceContent` spécifique, fournissent motivation/maîtrise de langue, et doivent être approuvés (`user_projects.approved = true`) avant édition.

**FAIT :** `TranslationProofreadingsController` exige authentification pour edit/update et vérifie l'accès ressource.

**À COMPRENDRE MAINTENANT :** Les utilisateurs non autorisés peuvent parcourir et télécharger (avec login pour certains fichiers). Éditer le contenu sacré exige un accès projet approuvé ou un rôle admin.

---

## Intégrité du pipeline d'import

Les données externes entrent via des importeurs dans `lib/importer/`, créant généralement des **drafts d'abord**, pas des écrasements directs.

### Correspondance d'identité pendant l'import

**FAIT :** L'importeur de traduction QuranEnc (`lib/importer/quran_enc.rb`) fait correspondre les données externes aux versets QUL par clé :

```ruby
def verses_by_key
  @verses_by_key ||= Verse.all.index_by(&:verse_key)
end

# For each imported row:
verse = verses_by_key["#{data['sura']}:#{data['aya']}"]
```

**FAIT :** Si `verse` est nil (clé non trouvée), l'import échouerait à `verse.verse_key` — il n'y a pas de repli silencieux vers une autre ayah.

**FAIT :** Les importeurs journalisent les problèmes plutôt que de continuer silencieusement pour certaines classes d'erreur :

```ruby
log_issue({ tag: 'missing-footnote-mapping', text: verse.verse_key })
log_issue({ tag: 'wrong-footnote-mapping', text: verse.verse_key })
```

**FAIT :** Les problèmes d'import créent des enregistrements `AdminTodo` pour suivi mainteneur.

**FAIT :** `Importer::Base#create_draft_tafsir` définit `text_matched` en comparant le draft au texte publié existant — rendant les écrasements involontaires visibles dans les files de relecture.

### Import en masse désactive le versioning

**FAIT :** `DraftContent::ApproveDraftContentJob` définit `PaperTrail.enabled = false` pendant l'import en masse, puis exécute `run_after_import_hooks` qui vérifie les enregistrements manquants.

### Hooks de validation post-import

**FAIT :** `ResourceContent#run_after_import_hooks` après approbation :

```ruby
update_records_count
set_meta_value('last-import-at', ...)
check_for_missing_translation  # expects exactly 6236 rows
check_for_missing_tafsirs      # expects coverage for every verse
```

**FAIT :** `check_for_missing_translation` :

```ruby
if !(6236 - translations.size).zero?
  issues.push("#{6236 - translations.size} missing translation record...")
end
if (missing_text = translations.where(text: [nil, ''])).any?
  issues.push "#{missing_text.size} translation with missing text..."
end
```

Ceux-ci retournent des tableaux de problèmes — ils ne bloquent pas l'import, mais les problèmes sont rapportés via commentaires `AdminTodo`.

---

## Outils d'intégrité des données

Au-delà des hooks d'import, QUL dispose d'outils explicites de contrôle d'intégrité.

### Page Admin Data Integrity Check

**FAIT :** `Tools::DataIntegrityChecks` (`app/models/tools/data_integrity_checks.rb`) fournit ~25 contrôles exécutables depuis Active Admin, incluant :

| Contrôle | Ce qu'il détecte |
|---|---|
| `words_without_root` | Mots sans assignation racine |
| `words_without_lemma` | Mots sans lemme |
| `words_without_stem` | Mots sans stem |
| `words_with_missing_arabic_text` | Colonnes script vides |
| `ayah_with_missing_translations` | Lacunes couverture traduction |
| `words_with_missing_translations` | Lacunes traduction niveau mot |
| `ayah_with_missing_tafsirs` | Lacunes couverture tafsir |
| `compare_translations` | Écarts entre éditions de traduction |
| `duplicate_mushaf_words` | Entrées layout dupliquées |
| `mushaf_words_with_incorrect_position` | Erreurs position layout |
| `ayah_with_different_mushaf_page` | Même ayah sur pages différentes entre mushafs |
| `ayah_without_matching_ayahs` | Lacunes références ayahs similaires |

**FAIT :** Ce sont des **requêtes diagnostiques**, pas des gates CI. Ils aident les mainteneurs à trouver des problèmes ; ils ne bloquent pas automatiquement les exports.

### Préservation des clés d'export

**FAIT :** Les exports de traduction utilisent `verse_key` comme clé d'objet JSON :

```ruby
json_data[translation.verse_key] = translation_text_without_footnotes(translation)
```

**FAIT :** Les exports script mot utilisent `location` comme clé et incluent des champs explicites `surah`, `ayah`, `word`.

**INFÉRENCE :** Tant que `verse_key` et `location` sont corrects au moment de l'export, l'intégrité de jointure en aval est préservée. L'export ne re-dérive pas les clés depuis les FK — il lit les champs chaîne dénormalisés.

---

## Comment la corruption se propage en aval

Comprendre les modes de défaillance vous aide à savoir quoi vérifier :

### Mauvais `ayah_number` ou `verse_key`

```text
Editor sets translation.verse_key = "2:256" (should be "2:255")
  → Export keys JSON as "2:256"
  → Downstream app joins translation to ayah 256
  → User sees wrong translation at wrong ayah
  → No error thrown; data looks structurally valid
```

**Atténuation :** Les champs d'identité sont re-synchronisés depuis `verse` pendant `Draft::Translation#import!`. Les modifications admin directes de `verse_key` sans passer par import sont le chemin de risque.

### Mauvais `word_position` ou `location`

```text
Morphology attached to word position 5 instead of 4
  → Export includes root/POS at wrong word
  → Word-by-word app shows wrong grammar for that token
```

**Atténuation :** `location` est dérivé de `word.location` pendant les exports ; les contrôles d'intégrité détectent racines/lemmes manquants.

### Mauvais `resource_content_id`

```text
Translation row assigned to wrong ResourceContent
  → Export for "Sahih International" includes another translator's text
  → Attribution broken; content may be correct Arabic/English but wrong source
```

**Atténuation :** L'UI admin scope par ressource ; les exporteurs filtrent `where(resource_content_id: id)`.

### Lignes manquantes (couverture incomplète)

```text
Translation resource has 6200 rows instead of 6236
  → run_after_import_hooks reports "36 missing translation record"
  → Downstream app gets nil for missing ayahs
```

**Atténuation :** `check_for_missing_translation` attend exactement 6236 lignes.

### Fichiers d'export obsolètes

```text
Content updated in Quran DB but export not refreshed
  → Public download serves old JSON
  → Downstream users see outdated text despite CMS showing new content
```

**Atténuation :** `DownloadableResource#refresh_export!` doit être exécuté après modifications de contenu. Re-export est une étape manuelle/déclenchée admin (souvent via job en arrière-plan).

---

## Contributions code vs contributions données

QUL supporte explicitement les deux chemins. Ils diffèrent en risque et attentes de relecture.

| Aspect | Contribution code | Contribution données |
|---|---|---|
| **Mécanisme** | Fork → branche → PR sur GitHub | Édition CMS, UI relecture, jobs import/approve |
| **Ce qui change** | App Rails, exporteurs, outillage, docs | Traductions, tafsirs, segments audio, morphologie |
| **Relecture** | Code review PR par mainteneurs | Relecture draft, approbation admin, porte accès projet |
| **Preuve nécessaire** | Tests, lint, diff ciblé | Mapping ayah/mot correct, UTF-8, comptages couverture |
| **Risque** | Bugs app, ruptures format export | Mauvais attachement ayah, texte sacré corrompu |
| **Versioning** | Historique Git | PaperTrail + tables draft + ChangeLog |
| **Rollback** | `git revert` | Revert PaperTrail dans admin, ou re-import depuis draft |

**FAIT :** `contribute-data.md` dit que la plupart de la contribution de données se fait via le CMS avec relecture avant publication sur `/resources`.

**FAIT :** Les contributions de données ne passent typiquement pas par des PR GitHub pour les lignes de contenu — elles passent par le pipeline draft → approve → export de l'application.

**INFÉRENCE :** En tant que contributeur code, vous pourriez construire/corriger les pipelines. En tant que contributeur données, vous travaillez à l'intérieur. Ne commitez pas de modifications de texte coranique en fichiers plats dans le dépôt sauf si explicitement partie d'un fixture importeur ou workflow documenté de contribution données.

---

## Ce qui N'EST PAS validé (lacunes à connaître)

**FAIT :** Les modèles de contenu centraux (`Translation`, `Tafsir`, `Verse`, `Word`) n'ont **pas de validations ActiveRecord** sur le contenu texte ou les champs d'identité.

**FAIT :** Imports et approbations utilisent fréquemment `save(validate: false)`.

**FAIT :** Les contrôles d'intégrité existent mais sont des **diagnostics opt-in**, pas des gates CI automatisées sur chaque modification.

**FAIT :** Il n'y a pas de test automatique affirmant que chaque ligne d'export mappe à la bonne ayah — cela devrait faire partie de votre vérification en modifiant les exporteurs.

| RÉALITÉ DU PROJET ACTUEL | BONNE PRATIQUE POUR VOS MODIFICATIONS |
|---|---|
| Comptages hooks post-import (6236 ayahs) | Ajouter tests ciblés en modifiant la logique de jointure export |
| Outils contrôle intégrité admin | Exécuter contrôles pertinents après modifications données |
| PaperTrail pour revert manuel | Documenter ce que vous avez vérifié dans la description PR |
| Draft/relecture pour éditions communautaires | Ne jamais contourner la relecture pour modifications de contenu |

---

## Checklist de vérification d'intégrité (pour vous plus tard)

Quand vous modifiez quoi que ce soit touchant les données coraniques, demandez :

### Contrôles d'identité
- [ ] Chaque ligne a-t-elle toujours le bon `verse_key` ou `location` ?
- [ ] Pour traductions : exactement 6236 lignes par ressource complète ?
- [ ] Pour ressources mot : `word_id` résout-il vers la `position` attendue ?

### Contrôles de contenu
- [ ] UTF-8 préservé ? Texte arabe non corrompu ?
- [ ] Échantillon 5–10 ayahs comparé manuellement avant/après ?
- [ ] Footnotes toujours liées aux bons IDs ?

### Contrôles d'export
- [ ] `refresh_export!` exécuté après modification de contenu ?
- [ ] Les clés JSON exportées correspondent-elles à `verse_key` / `location` ?
- [ ] Comptages lignes SQLite conformes aux attentes ?

### Contrôles de provenance
- [ ] `resource_content_id` inchangé pour ressources non affectées ?
- [ ] Version source `meta_data` mise à jour si re-importé ?
- [ ] `ChangeLog` rédigé si les utilisateurs doivent connaître la mise à jour ?

---

## Classification pour cette phase

### À COMPRENDRE MAINTENANT

1. **Champs d'identité** (`verse_key`, `location`, `position`) sont infrastructure. **Champs de contenu** (`text`, `text_*`) sont éditoriaux.
2. **Draft → approve → import** est le chemin normal de modification de contenu. Les éditions communautaires n'écrasent pas directement le texte publié.
3. **PaperTrail** versionne les modifications de contenu et supporte revert dans admin. Les imports en masse le désactivent temporairement.
4. **`ResourceContent`** porte la provenance : auteur, source de données, `meta_data`, `approved`.
5. **`DownloadableResource.published`** contrôle les téléchargements publics. Le contenu peut exister en base sans être exporté publiquement.
6. **Les imports correspondent par `verse_key`** — externe `sura:aya` → `Verse.verse_key`. Pas de validations sur modèles centraux ; contrôles d'intégrité sont des outils séparés.
7. **De mauvais identifiants se propagent silencieusement** dans les exports. Une réponse HTTP réussie ne suffit pas à prouver l'intégrité des données.

### UTILE PLUS TARD

- Enums copyright/hébergement `ResourcePermission`
- `ChangeLog` comme annonces de mise à jour orientées utilisateur
- `AdminTodo` pour suivi problèmes d'import
- Workflow `Morphology::Phrase#approved`
- Flux demande accès contributeur `UserProject`
- `Audio::ChangeLog` pour historique mises à jour récitation
- `Tools::TajweedRulesCheck` (séparé des contrôles intégrité données)

### IGNORER POUR L'INSTANT

- `LogEntry` (analytics, pas gouvernance contenu)
- `ProofReadComment` (commentaires sur ressources, secondaire au workflow draft)
- Modèle `Contributor` (page crédits site, pas attribution par ressource)
- Modèles staging `RawData::*`
- Implémentations parseur importeur individuelles (Phase 10)

---

## Incertitudes

| Élément | Statut |
|---|---|
| Quelle base PaperTrail stocke versions pour modèles QuranApiRecord | **INCONNU** — vérifier localement en Phase 17 |
| Si `approved` sur `ResourceContent` bloque export ou filtre seulement scopes | **PARTIELLEMENT CONNU** — `ExportDbForSemanticSearchJob` filtre `approved` ; pas tous les exporteurs |
| Liste complète clés `meta_data` par type ressource | **INCONNU** — inspecter enregistrements par sub_type |
| Si suggestions relecture communautaire notifient auto les relecteurs | **INCONNU** — probablement relecture manuelle dans admin |
| Gates CI intégrité données | **FAIT : aucun trouvé** — contrôles sont outils admin uniquement |

---

## Ce que nous investiguerons ensuite

**Phase 4 — Rails pour un ingénieur React/Node**

Maintenant que vous comprenez les données et leur gouvernance, nous cartographions l'usage Rails de QUL vers des concepts que vous connaissez déjà : routes, controllers, Active Record, Active Admin, Sidekiq, etc. — seulement ce qui est réellement présent dans ce codebase.

---

## Résumé Phase 3 — cinq choses à retenir

1. **L'identité est infrastructure ; le contenu est éditorial.** Protégez `verse_key`, `location` et `position` avant tout.
2. **Les modifications suivent draft → relecture → import → export → publication.** Les suggestions communautaires n'écrasent pas directement le contenu Coran publié.
3. **La provenance vit sur `ResourceContent`** — auteur, source de données, `meta_data`, permissions — pas sur chaque ligne ayah.
4. **PaperTrail + outils contrôle intégrité** sont le filet de sécurité — pas les validations ActiveRecord (largement absentes sur modèles contenu).
5. **De mauvaises jointures se propagent silencieusement aux exports.** Vérifiez identifiants et échantillonnez données ; ne faites pas confiance à un save réussi seul.

---

*Généré pendant l'onboarding contributeur QUL. Phase 3 sur ~24. Investigation en lecture seule — aucun code modifié.*

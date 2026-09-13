# Phase 2 — Modèle de données coranique d'abord

> **Série d'onboarding :** plongée progressive pour devenir un contributeur QUL légitime.  
> **Prérequis :** [Phase 1 — Qu'est-ce que QUL ?](phase-01-what-is-qul.md)  
> **Ce fichier :** les données elles-mêmes — hiérarchie, identifiants et comment les ressources s'attachent. Toujours pas une visite de l'architecture Rails.

---

## Pourquoi commencer par les données, pas Rails

Avant que `belongs_to` et `has_many` signifient quoi que ce soit, vous devez savoir **ce qui est joint**.

Le problème technique central de QUL est :

> De nombreux packages de ressources indépendants (traductions, morphologie, audio, …) doivent tous se référer aux **mêmes ayahs et mots** de manière fiable.

Tout le reste — formulaires CMS, exporteurs, jobs en arrière-plan — existe pour créer, protéger et publier ces données joignables.

---

## La hiérarchie (conceptuelle)

```text
Quran
 └── Surah (114 chapters)
      └── Ayah (verses within a surah)
           └── Word (tokens within an ayah)
```

Ce n'est pas qu'une taxonomie. C'est le **schéma d'adressage** que chaque jeu de données utilise.

---

## Noms de modèles Rails vs noms courants

Le codebase de QUL utilise la terminologie arabe/corane de façon incohérente entre les couches. Apprenez ce tableau tôt :

| Concept | Docs courantes / export | Modèle Rails | Table principale | Champs clés |
|---|---|---|---|---|
| Surah | `surah`, `surah_id` | `Chapter` | `chapters` | `chapter_number` (1–114), `id` |
| Ayah | `ayah`, `ayah_number` | `Verse` | `verses` | `verse_number`, `chapter_id`, `verse_key` |
| Word | `word`, `word_position` | `Word` | `words` | `position`, `verse_id`, `location` |

**FAIT :** `Chapter#name` est aliasé à `id` — donc `chapter.name` retourne le numéro de sourate, pas un nom d'affichage. Les noms d'affichage utilisent `name_simple`, `name_arabic`, etc.

**FAIT :** `Verse#to_s` retourne `verse_key` (ex. `"2:255"`).

**FAIT :** `Word#to_s` retourne `location` (ex. `"2:255:13"`).

---

## Aide-mémoire des identifiants

Trois couches d'identifiants existent. Elles servent des objectifs différents.

### 1. Clés composites lisibles par l'humain (préférées pour les jointures)

| Clé | Format | Exemple | Utilisé pour |
|---|---|---|---|
| `verse_key` | `surah:ayah` | `"2:255"` | Ressources au niveau ayah |
| `location` | `surah:ayah:word` | `"2:255:13"` | Ressources au niveau mot |

**FAIT :** `Utils::Quran.get_ayah_key(surah, ayah)` retourne `"#{surah}:#{ayah}"`.

**FAIT :** Le code d'export découpe `word.location` en composants `surah`, `ayah`, `word` (`lib/exporter/export_quran_word_script.rb`).

### 2. Clés étrangères relationnelles (dans la base de données)

| Champ | Pointe vers | Exemple |
|---|---|---|
| `chapter_id` | `chapters.id` | Sourate 2 → `chapter_id: 2` |
| `verse_id` | `verses.id` | FK vers la ligne ayah |
| `word_id` | `words.id` | FK vers la ligne mot |
| `verse_number` | ayah dans la sourate | `255` (avec `chapter_id: 2`) |
| `position` | mot dans l'ayah | `13` (avec `verse_id`) |

**FAIT :** La plupart des tables de contenu (`translations`, `tafsirs`, `word_translations`) stockent **à la fois** des FK (`verse_id`, `chapter_id`) **et** des clés dénormalisées (`verse_key`, `verse_number`) pour des requêtes et exports efficaces.

### 3. Indices séquentiels globaux (navigation et exports compacts)

| Champ | Modèle | Plage | Rôle |
|---|---|---|---|
| `verse_index` | `Verse` | 1–6236 | Séquence d'ayah globale dans tout le Coran |
| `word_index` | `Word` | 1–~77k | Séquence de mots globale ; **index unique** |
| `sequence_number` | `Word` | — | ID global de mot également unique |

**FAIT :** `Verse#next_ayah` navigue via `verse_index + 1`, pas `id + 1`.

**UTILE PLUS TARD :** Du code mushaf/layout stocke des valeurs `verse_index` dans des colonnes nommées `first_verse_id` / `last_verse_id` sur `MushafPage`. **INFÉRENCE :** C'est une incohérence de nommage — ces champs se comportent comme `verse_index`, pas `verses.id`. Quand vous voyez `*_verse_id` dans le code layout, vérifiez quel espace d'ID est visé.

### Variantes de noms de champs d'export

La documentation (`data-model.md`) indique aux développeurs en aval :

```text
surah_id + ayah_number           → ayah-level joins
surah_id + ayah_number + word_position → word-level joins
```

Les exports utilisent des noms variés :

| Nom docs | Également vu comme |
|---|---|
| `surah_id` | `surah`, `chapter_id`, `chapter_number` |
| `ayah_number` | `ayah`, `verse_number` |
| `word_position` | `word`, `position` |

**À COMPRENDRE MAINTENANT :** Normalisez ces variantes mentalement. Elles se réfèrent à la même hiérarchie.

---

## Exemple travaillé : Ayat al-Kursi (2:255)

```text
Chapter (Surah 2 — Al-Baqarah)
  chapter_number: 2
  id: 2
  │
  └── Verse (Ayah 255)
        verse_key: "2:255"
        verse_number: 255
        chapter_id: 2
        verse_index: 262  (global position in Quran — illustrative; verify against dump)
        │
        ├── text_uthmani: "اللَّهُ لَا إِلَٰهَ إِلَّا هُوَ ..."
        ├── text_qpc_hafs: (another script column)
        ├── text_indopak: (another script column)
        │     ... many script columns on one row
        │
        ├── Translation (resource_content_id: 131, e.g. Sahih International)
        │     verse_key: "2:255"
        │     text: "Allah - there is no deity except Him..."
        │
        ├── Tafsir (resource_content_id: X)
        │     verse_key: "2:255"
        │     start_verse_id..end_verse_id (may span multiple ayahs)
        │
        └── Words
              ├── Word position 1
              │     location: "2:255:1"
              │     text_uthmani: "اللَّهُ"
              │     root_id → Root
              │     lemma_id → Lemma
              │     └── Morphology::Word (grammar analysis)
              │
              ├── Word position 2
              │     location: "2:255:2"
              │     ...
              └── ... (this ayah has many words; not all positions are "words")
```

**FAIT :** Une seule ligne `Verse` contient **plusieurs représentations de script arabe** en colonnes séparées (`text_uthmani`, `text_indopak`, `text_qpc_hafs`, `text_imlaei`, …). Ce sont des variantes de contenu sur l'ayah canonique, pas des identités d'ayah séparées.

**FAIT :** Une seule ligne `Word` contient également plusieurs colonnes de script. Quelle colonne un export utilise est déterminé par les métadonnées `ResourceContent` (`text_type`).

---

## L'épine dorsale : Chapter → Verse → Word

### Chapter (table `chapters`, modèle `Chapter`)

**FAIT** — champs clés :

| Champ | Signification |
|---|---|
| `chapter_number` | Numéro de sourate 1–114 |
| `name_simple` | Nom anglais (« Al-Baqarah ») |
| `name_arabic` | Nom arabe |
| `revelation_place` | `"makkah"` ou `"madinah"` |
| `verses_count` | Nombre d'ayahs dans cette sourate |

**Associations :** `has_many :verses`, `has_many :chapter_infos`

### Verse (table `verses`, modèle `Verse`)

**FAIT** — champs clés :

| Champ | Signification |
|---|---|
| `chapter_id` | Sourate parente |
| `verse_number` | Numéro d'ayah dans la sourate |
| `verse_key` | Chaîne `"surah:ayah"` |
| `verse_index` | Index d'ayah global (1–6236) |
| `words_count` | Nombre de tokens mot dans cette ayah |
| `juz_number`, `hizb_number`, `page_number`, … | Métadonnées structurelles / navigation |
| Colonnes `text_*` | Texte arabe en divers scripts |

**Associations (sélection) :**

```ruby
belongs_to :chapter
has_many :words
has_many :translations
has_many :tafsirs
has_many :morphology_words, class_name: 'Morphology::Word'
has_many :actual_words, -> { where char_type_id: 1 }, class_name: 'Word'
```

**FAIT :** `actual_words` filtre sur `char_type_id: 1` — toutes les lignes dans `words` ne sont pas des « mots » au sens linguistique. Certaines positions sont des marqueurs (waqf, sajdah, etc.). Le scope `Word.words` applique le même filtre.

### Word (table `words`, modèle `Word`)

**FAIT** — champs clés :

| Champ | Signification |
|---|---|
| `verse_id` | Ayah parente |
| `chapter_id` | Référence sourate dénormalisée |
| `position` | Position du mot dans l'ayah (base 1) |
| `location` | Chaîne `"surah:ayah:position"` |
| `word_index` | Index de mot global |
| `char_type_id` | Mot vs marqueur vs autres types de token |
| `root_id`, `lemma_id`, `stem_id` | Références linguistiques |
| Colonnes `text_*` | Texte arabe en divers scripts |

**Associations (sélection) :**

```ruby
belongs_to :verse
belongs_to :root, :lemma, :stem  # optional
has_many :word_translations
has_one :morphology_word, class_name: 'Morphology::Word'
has_many :mushaf_words
```

**INFÉRENCE :** `root_id`/`lemma_id`/`stem_id` sur `Word` sont les liens linguistiques de « recherche rapide ». L'analyse morphologique plus profonde vit dans `Morphology::Word` et les tables associées.

---

## ResourceContent : l'enveloppe de « package logique »

C'est le deuxième modèle le plus important après la hiérarchie elle-même.

Une **ressource** dans QUL n'est pas une seule ligne de base de données. C'est un **package logique** identifié par `ResourceContent` :

```text
ResourceContent (id: 131, name: "Sahih International", sub_type: "translation")
  ├── metadata: author, language, data_source, approved, cardinality_type
  ├── many Translation rows (one per ayah), each with resource_content_id: 131
  ├── FootNote rows (attached to translations)
  ├── ChangeLog entries (CMS DB — see Phase 5)
  └── DownloadableResource (public listing) + DownloadableFiles (exported JSON/SQLite)
```

**FAIT :** `ResourceContent` vit dans la base de contenu Coran (`QuranApiRecord`).

**FAIT :** Champs de métadonnées clés :

| Champ | Rôle |
|---|---|
| `name` | Nom d'affichage (« Sahih International ») |
| `sub_type` | Type : `translation`, `tafsir`, `recitation`, `morphology`, `quran-script`, … |
| `cardinality_type` | Granularité : `1_ayah`, `1_word`, `1_chapter`, `quran`, … |
| `language_id` | Langue de la ressource |
| `author_id` | Attribution traducteur/érudit |
| `data_source_id` | Origine des données |
| `approved` | Si la ressource est approuvée pour usage |
| `meta_data` (jsonb) | Config flexible : `text-type`, `has-footnote`, `has-segments`, clés source, etc. |

**FAIT :** `ResourceContent` définit des constantes de cardinalité :

```ruby
CardinalityType::OneVerse  = '1_ayah'    # one row per ayah
CardinalityType::OneWord   = '1_word'     # one row per word
CardinalityType::OneChapter = '1_chapter' # one row per surah
CardinalityType::Quran     = 'quran'      # whole-quran resource
```

**FAIT :** `ResourceContent` définit des constantes de sous-type incluant : `translation`, `tafsir`, `transliteration`, `recitation`, `morphology`, `quran-script`, `layout`, `topic`, `theme`, `mutashabihat`, `font`, `meta`.

### La concern Resourceable

La plupart des lignes de contenu incluent `Resourceable` :

```ruby
belongs_to :resource_content, optional: true
```

Cela signifie qu'une `Translation`, `Tafsir`, `WordTranslation`, `Audio::Recitation`, etc. pointent toutes vers leur package parent via `resource_content_id`.

**À COMPRENDRE MAINTENANT :** Quand vous voyez 6 236 lignes `Translation`, ce ne sont pas 6 236 « ressources » séparées. Ce sont des lignes appartenant à **un seul** `ResourceContent` (une édition de traduction).

---

## Comment chaque type de ressource s'attache

### Ressources au niveau ayah

#### Traductions (table `translations`)

**FAIT :**

```ruby
class Translation < QuranApiRecord
  belongs_to :verse, optional: true
  belongs_to :resource_content  # which translation edition
  has_many :foot_notes
  # verse_key, chapter_id, verse_number denormalized on each row
end
```

| Jointure | Clés |
|---|---|
| Vers ayah | `verse_id` OU `verse_key` OU `chapter_id + verse_number` |
| Vers package ressource | `resource_content_id` |

Un `ResourceContent` → ~6 236 lignes `Translation` (une par ayah).

#### Tafsirs (table `tafsirs`)

**FAIT :** Similaire aux traductions, mais peut couvrir des **plages d'ayahs** :

| Champ | Rôle |
|---|---|
| `verse_id` | Ayah principale |
| `start_verse_id`, `end_verse_id` | Plage couverte par cette entrée tafsir |
| `group_verse_key_from`, `group_verse_key_to` | Plage lisible par l'humain |
| `group_tafsir_id` | Lie les entrées groupées |

**FAIT :** `Tafsir.for_verse(verse, resource)` trouve le tafsir où `verse.id` tombe entre `start_verse_id` et `end_verse_id`.

**INFÉRENCE :** La logique de jointure tafsir est plus complexe que la traduction — on ne peut pas supposer un mapping ayah strict 1:1.

#### Translittérations (table `transliterations`)

**FAIT :**

```ruby
belongs_to :resource, polymorphic: true  # can attach to Verse or Word
belongs_to :resource_content
```

Polymorphe — peut être au niveau ayah ou mot selon ce à quoi il s'attache.

#### Translittérations arabes (table `arabic_transliterations`)

**FAIT :** Modèle séparé de `Transliteration`. S'attache à la fois à `verse` et `word`. Utilisé pour les overlays de translittération visuelle style indopak.

#### Packages script Coran (`quran_script_by_verses`, `quran_script_by_words`)

**FAIT :** Les exports de script peuvent être organisés par ayah (`QuranScript::ByVerse`) ou par mot (`QuranScript::ByWord`), chaque ligne liée à un `resource_content_id` et sélectionnant quelle colonne `text_*` exporter.

#### Sujets et thèmes

| Modèle | Niveau de jointure | Mécanisme |
|---|---|---|
| `VerseTopic` | Ayah | `verse_id` + `topic_id` |
| `AyahTheme` | Plage d'ayahs | `verse_id_from..verse_id_to` |

#### Info chapitre (table `chapter_infos`)

**FAIT :** Jointure au niveau sourate : `chapter_id` + `resource_content_id` + `language_id`.

#### Audio (`audio_recitations`, `audio_files`, `audio_segments`)

```text
Audio::Recitation (one reciter's package, resource_content_id)
  └── Audio::ChapterAudioFile (one surah's audio file)
        └── Audio::Segment (per-ayah timing data)
              verse_id, verse_key, timestamp_from, timestamp_to, segments (jsonb)
```

**FAIT :** `Audio::Segment` stocke `verse_key`, `verse_id` et des timestamps en millisecondes. Le JSON de segments mappe les positions de mots aux limites de timing.

### Ressources au niveau mot

#### Traductions de mots (table `word_translations`)

**FAIT :**

```ruby
belongs_to :word
belongs_to :resource_content
# text = translated gloss for this word
# group_word_id, group_text = multi-word phrase translations
```

Jointure : `word_id` (→ `word.location` → `surah:ayah:position`).

**FAIT :** Supporte les **traductions groupées** où une glose couvre plusieurs mots (`word_range_from..word_range_to`).

#### Morphologie (`morphology_words` et tables associées)

```text
Word (canonical Arabic token)
  ├── root_id → Root (مادة)
  ├── lemma_id → Lemma (مدخل)
  ├── stem_id → Stem
  └── Morphology::Word (deeper grammatical analysis)
        ├── grammar_pattern
        ├── word_segments (morphological segments)
        ├── word_tokens
        ├── derived_words
        └── grammar_concepts (through word_grammar_concepts)
```

**FAIT :** `Morphology::Word` lie à la fois `word_id` et `verse_id`, et porte `location` (dénormalisé).

**FAIT :** Tables de dictionnaire linguistique :

| Modèle | Table | Rôle |
|---|---|---|
| `Root` | `roots` | Racine trilitère (`text_uthmani`, `value`) |
| `Lemma` | `lemmas` | Forme d'entrée de dictionnaire |
| `Stem` | `stems` | Forme de racine |
| `Token` | `tokens` | Tokenisation supplémentaire |

Ce sont des **données de référence partagées** — de nombreux mots pointent vers la même racine/lemme.

#### Layout mushaf (`mushafs`, `mushaf_pages`, `mushaf_words`)

**FAIT :** C'est là que le **contenu rencontre le layout** :

| Modèle | Rôle |
|---|---|
| `Mushaf` | Une édition de layout (ex. Madani 15 lignes) |
| `MushafPage` | Une page : quels versets/mots apparaissent |
| `MushafWord` | Position d'un mot sur une page : `page_number`, `line_number`, `position_in_line`, `text` |

**FAIT :** `MushafWord` appartient à la fois à `word` (contenu canonique) et `mushaf` (édition de layout). Le même `Word` peut apparaître dans plusieurs layouts mushaf avec des positions page/ligne différentes.

**UTILE PLUS TARD :** Le mushaf est une préoccupation séparée de la hiérarchie centrale. La Phase 13 ira plus loin. Pour l'instant : le **contenu** vit sur `Word` ; le **layout** vit sur `MushafWord`.

---

## Contenu sur l'épine dorsale vs tables de ressources séparées

Cette distinction compte pour comprendre ce qui est « le Coran » vs ce qui est « une ressource attachée au Coran ».

### Contenu SUR l'épine dorsale (colonnes sur Verse/Word)

| Emplacement | Quoi |
|---|---|
| `verses.text_*` | Texte arabe ayah en plusieurs scripts |
| `words.text_*` | Texte arabe mot en plusieurs scripts |
| `words.root_id`, `lemma_id`, `stem_id` | Liens linguistiques centraux |

**INFÉRENCE :** Ce sont les données les plus sensibles. Les modifier affecte toute ressource qui référence cette ayah ou ce mot.

### Contenu ATTACHÉ via tables de ressources

| Table | Attaché à | Scopé par |
|---|---|---|
| `translations` | ayah | `resource_content_id` |
| `tafsirs` | ayah (ou plage) | `resource_content_id` |
| `word_translations` | mot | `resource_content_id` |
| `morphology_words` | mot | `resource_content_id` optionnel |
| `transliterations` | ayah ou mot (polymorphe) | `resource_content_id` |
| `audio_segments` | ayah | `audio_recitation_id` |

**À COMPRENDRE MAINTENANT :** L'épine dorsale (`Chapter` → `Verse` → `Word`) est relativement stable. Les tables de ressources se multiplient vers l'extérieur — de nombreuses traductions, récitations, analyses morphologiques, toutes accrochées aux mêmes adresses.

---

## Chaîne de publication : de la ligne de contenu au téléchargement

```text
Content rows (Translation, etc.)
  └── ResourceContent (logical package, metadata, approval)
        └── DownloadableResource (public catalog entry, published: true)
              └── DownloadableFile (actual JSON/SQLite file attachment)
```

**FAIT :**

- `DownloadableResource` est dans la **base CMS** (`ApplicationRecord`).
- Il référence `resource_content_id` qui pointe dans la **base Coran**.
- `DownloadableFile` a une pièce jointe Active Storage (`file`) contenant l'artefact exporté.

**INFÉRENCE :** La publication est un concept en deux étapes : le contenu doit exister dans la base Coran **et** un `DownloadableResource` doit être marqué `published: true` avec des fichiers générés attachés.

(Nous tracerons ce workflow en Phases 8–11.)

---

## Données draft et relecture (base CMS)

Toutes les modifications ne vont pas directement aux tables de contenu. La base CMS contient l'état draft/relecture :

| Table CMS | Rôle |
|---|---|
| `draft_translations` | Modifications de traduction suggérées en attente de relecture |
| `draft_tafsirs` | Modifications de tafsir suggérées |
| `draft_word_translations` | Modifications de traduction de mot suggérées |
| `draft_contents` | Contenu draft générique |
| `draft_foot_notes` | Modifications de notes de bas de page draft |

**FAIT :** `Translation#save_suggestions` crée un `Draft::Translation` avec `need_review: true` plutôt que d'écraser immédiatement le texte publié.

**INFÉRENCE :** Le pattern est : **suggérer → relire → approuver → écrire dans la table de contenu Coran**. Cela protège les données publiées.

(La Phase 3 tracera provenance et versioning en détail.)

---

## Frontière inter-bases (aperçu)

**FAIT :** La plupart des modèles de contenu Coran héritent de `QuranApiRecord` → connexion à `quran_dev` / `quran_api_db`.

**FAIT :** Les modèles CMS (`User`, `DownloadableResource`, `Draft::Translation`, `Morphology::Phrase`, …) héritent de `ApplicationRecord` → connexion à `quran_community_tarteel`.

**FAIT :** `db/schema.rb` ne contient que le schéma de la **base CMS**. Les tables de contenu Coran ne sont **pas** dans ce fichier — elles viennent du dump SQL (`mini_quran_dev.sql`).

**FAIT :** Certains modèles CMS stockent des références entières vers des lignes de la base Coran (ex. `Morphology::Phrase#source_verse_id`, tables draft avec `verse_id`). Ce sont des **clés étrangères logiques** — Rails ne peut pas imposer des associations inter-bases.

**À COMPRENDRE MAINTENANT :** Vous verrez `verse_id` sur des tables dans les deux bases. Elles se réfèrent à la même ayah conceptuelle, mais seul `Verse` dans la base Coran est la ligne autoritaire.

(La Phase 5 couvre cette architecture complètement.)

---

## Diagramme entité-relation

```mermaid
erDiagram
    Chapter ||--o{ Verse : "has many"
    Verse ||--o{ Word : "has many"
    Word }o--o| Root : "root_id"
    Word }o--o| Lemma : "lemma_id"
    Word }o--o| Stem : "stem_id"

    ResourceContent ||--o{ Translation : "resource_content_id"
    ResourceContent ||--o{ Tafsir : "resource_content_id"
    ResourceContent ||--o{ WordTranslation : "resource_content_id"
    ResourceContent ||--o{ Audio_Recitation : "resource_content_id"

    Verse ||--o{ Translation : "verse_id"
    Verse ||--o{ Tafsir : "verse_id"
    Word ||--o{ WordTranslation : "word_id"
    Word ||--o| Morphology_Word : "word_id"

    ResourceContent ||--o| DownloadableResource : "resource_content_id"
    DownloadableResource ||--o{ DownloadableFile : "has files"

    Verse {
        int chapter_id
        int verse_number
        string verse_key
        int verse_index
        string text_uthmani
    }

    Word {
        int verse_id
        int position
        string location
        int word_index
        int root_id
    }

    Translation {
        int verse_id
        int resource_content_id
        string verse_key
        text text
    }

    ResourceContent {
        string name
        string sub_type
        string cardinality_type
        boolean approved
        jsonb meta_data
    }
```

---

## Pièges de nommage pour vous protéger

| Piège | Réalité |
|---|---|
| `chapter_id` signifie sourate | Oui, toujours |
| `verse_id` signifie toujours `verses.id` | Généralement, mais certains champs layout nommés `*_verse_id` stockent `verse_index` |
| `resource_content_id` vs `resource_id` | `resource_content_id` est le package ; `resource_id` sur `ResourceContent` est un lien polymorphe vers un enregistrement principal (ex. un Mushaf) |
| `Word` signifie mot linguistique | Pas toujours — vérifiez `char_type_id`. Utilisez le scope `Word.words` ou l'association `actual_words` |
| `db/schema.rb` est le schéma complet | Base CMS uniquement. Les tables Coran viennent du dump SQL |
| L'export utilise `surah_id` | Le code peut utiliser `surah`, `chapter_id` ou `chapter_number` — même concept |

---

## Classification pour cette phase

### À COMPRENDRE MAINTENANT

1. **Chapter → Verse → Word** est l'épine dorsale. Tout s'y attache.
2. **`verse_key`** (`"2:255"`) et **`location`** (`"2:255:13"`) sont les adresses lisibles par l'humain.
3. **`ResourceContent`** est le package logique — une édition de traduction, une récitation, un jeu de morphologie.
4. Les lignes de contenu (`Translation`, `WordTranslation`, etc.) sont scopées par **`resource_content_id`**.
5. Les variantes de script arabe sont des **colonnes** sur `Verse`/`Word`, pas des identités d'ayah séparées.
6. L'attachement niveau ayah vs mot est déterminé par `cardinality_type` et le modèle utilisé.
7. `db/schema.rb` ≠ base complète. Le contenu Coran vit dans une base séparée chargée depuis le dump.

### UTILE PLUS TARD

- Regroupement tafsir par plage d'ayahs (`start_verse_id..end_verse_id`)
- Regroupement traduction de mots (`group_word_id`, `group_text`)
- Tables de grammaire profonde `Morphology::Word` (segments, tokens, mots dérivés)
- Famille de modèles layout mushaf (`Mushaf` → `MushafPage` → `MushafWord`)
- Filtrage `char_type_id` pour mots réels vs marqueurs
- Navigation globale `verse_index` / `word_index`
- Tables draft dans la base CMS pour workflows de relecture

### IGNORER POUR L'INSTANT

- Tables individuelles de concepts grammaticaux morphologiques
- `Morphology::Phrase` / correspondance de phrases mutashabihat (vit dans la base CMS)
- Variantes récitation gapless vs ayah par ayah
- Modèles de staging d'import `RawData::*`
- Détails des tables de métadonnées de navigation (juz, hizb, ruku, manzil)
- Encodage police/glyphe (`code_v1`, `code_v2`)

---

## Incertitudes

| Élément | Statut |
|---|---|
| Si `verses.id` égale toujours `verse_index` dans les données de production | **INCONNU** — le code utilise les deux ; vérifier contre le dump en Phase 17 |
| Valeurs exactes de `char_type_id` pour mot vs marqueur vs fin | **INCONNU** — interroger la table `char_types` localement |
| Quelles analyses morphologiques sont scopées ressource vs globales | **PARTIELLEMENT CONNU** — `Morphology::Word` a un `resource_content_id` optionnel ; racines/lemmes semblent partagés |
| Liste complète des clés `meta_data` par type de ressource | **INCONNU** — inspecter les enregistrements `ResourceContent` par sub_type |

---

## Ce que nous investiguerons ensuite

**Phase 3 — Données sacrées, provenance et intégrité**

Maintenant que vous savez *ce que* sont les données, nous traçons :

- Quels champs représentent l'identité canonique (immuable) vs le contenu éditable
- Flags `approved`, workflows draft, versioning PaperTrail
- `Author`, `DataSource`, `ChangeLog`, métadonnées de provenance
- Quelles validations protègent contre la corruption
- Comment une mauvaise modification se propage aux exports

---

## Résumé Phase 2 — cinq choses à retenir

1. **Chapter → Verse → Word** est l'épine dorsale. `verse_key` et `location` sont les adresses.
2. **`ResourceContent`** enveloppe un package logique — une édition de traduction, une récitation, etc.
3. Les lignes de contenu s'attachent à l'épine dorsale via `verse_id` ou `word_id`, scopées par `resource_content_id`.
4. Les scripts arabes sont des **colonnes** sur les lignes de l'épine dorsale, pas des identités séparées. Les tables de ressources pendent vers l'extérieur.
5. Deux bases existent — contenu Coran vs CMS. `db/schema.rb` ne montre que le CMS. Drafts et téléchargements vivent dans le CMS ; versets et traductions vivent dans la base Coran.

---

*Généré pendant l'onboarding contributeur QUL. Phase 2 sur ~24. Investigation en lecture seule — aucun code modifié.*

# Phase 9 — Base de données et recherche sémantique

> **Prérequis :** Phases [1](./phase-1-what-is-openhikmah.md)–[8](./phase-8-state-and-canvas.md)  
> **Étiquettes de preuve :** **FACT** (fait établi) · **INFERENCE** (inférence) · **UNKNOWN** (inconnu)

La phase 8 couvrait **l'état du canvas dans le navigateur**. La phase 9 couvre la **couche de persistance et de récupération** sous-jacente à la recherche, à la découverte ancrée et au graphe de connexions IA — au niveau ingénieur, pas comme un tutoriel SQL générique.

**Phrase centrale (à mémoriser) :**

> Le texte sacré du Coran et les vecteurs d'ancrage sont **chargés une fois dans Postgres** ; le code d'exécution **les lit**. Le classement sémantique repose sur **des calculs sur des vecteurs stockés**, pas sur la mémoire du LLM.

---

## PostgreSQL dans OpenHikmah

**FACT:** L'application utilise **PostgreSQL 16** avec l'extension **pgvector** (`docker-compose.yml` → `pgvector/pgvector:pg16`).

**FACT:** Connexion via `DATABASE_URL` → `lib/infra/db.ts` :

```typescript
export const db = drizzle(client, { schema });
```

Drizzle encapsule un client `postgres` (postgres-js) avec `max: 10` connexions.

### Niveaux de tables (ce qui compte pour les contributeurs)

```mermaid
flowchart TB
  subgraph tier1 ["Niveau 1 — Corpus sacré / ancrage (chargé hors ligne)"]
    V[(verses)]
    VT[(verse_translations)]
    VE[(verse_embeddings)]
    WM[(word_morphology)]
  end

  subgraph tier2 ["Niveau 2 — Graphe produit IA (écritures à l'exécution)"]
    C[(connections)]
    CC[(connection_coverage)]
    AG[(ai_generations)]
  end

  subgraph tier3 ["Niveau 3 — Instantanés utilisateur / session"]
    B[(bookmarks)]
    SW[(saved_workspaces)]
    SC[(shared_canvases)]
  end

  subgraph tier4 ["Niveau 4 — Ops / admin / analytics"]
    RL[(rate_limits)]
    SL[(search_log)]
    FF[(feature_flags)]
    PV[(prompt_versions)]
    JR[(job_runs)]
  end

  tier1 --> tier2
  tier2 --> Canvas[Canvas via API — pas de lecture DB directe]
  tier3 --> Canvas
```

| Niveau | Exemples | Qui écrit | Risque pour le contributeur |
| --- | --- | --- | --- |
| **1 — Corpus** | `verses`, `verse_embeddings`, `word_morphology` | Scripts de seed / jobs admin | **Élevé** — données sacrées + ancrage |
| **2 — Graphe** | `connections`, `ai_generations` | `graph-service` en cas de cache miss | **Élevé** — sortie IA théologique |
| **3 — Utilisateur** | `bookmarks`, `saved_workspaces` | Routes API auth | Moyen |
| **4 — Ops** | `rate_limits`, `search_log`, `feature_flags` | Infra / admin | Moyen–faible |

**À COMPRENDRE MAINTENANT :** Les phases 7–8 ont déjà séparé **`connections` Postgres** des **arêtes du canvas**. La phase 9 ajoute : **`verse_embeddings` et `word_morphology` sont les entrées déterministes** de `discoverCandidates` de la phase 7.

---

## Drizzle ORM — comment ce dépôt l'utilise

**FACT:** Source unique du schéma : `lib/infra/db/schema.ts` (~660 lignes, toutes les définitions `pgTable` + types `$inferSelect` exportés).

**FACT:** Configuration Drizzle Kit (`drizzle.config.ts`) :

```typescript
schema: "./lib/infra/db/schema.ts",
out: "./lib/infra/db/migrations",
dialect: "postgresql",
```

### Motifs que vous rencontrerez

| Motif | Exemple dans ce dépôt |
| --- | --- |
| Définition de table | `export const verses = pgTable("verses", { ... })` |
| Index | `uniqueIndex`, `index`, index partiels avec `.where(sql`...`)` |
| Colonne pgvector | `vector("embedding", { dimensions: 768 })` |
| Index HNSW | `.using("hnsw", t.embedding.op("vector_cosine_ops"))` |
| Requêtes | `db.select().from(verseEmbeddings).where(...).orderBy(desc(similarity))` |
| SQL brut si nécessaire | `db.execute(sql`TRUNCATE ...`)` dans les tests d'intégration |

**FACT:** Le code applicatif importe `db` depuis `@/lib/infra/db` et les objets table depuis `@/lib/infra/db/schema`.

**INFERENCE:** Contrairement au modèle document de Firestore, OpenHikmah privilégie **des tables relationnelles + des index explicites** pour les refs de versets, les cellules du graphe et la recherche ANN vectorielle.

---

## Migrations

**FACT:** Les migrations SQL se trouvent dans `lib/infra/db/migrations/` (numérotées `0000` … `0023`+).

| Commande / script | Rôle |
| --- | --- |
| `node scripts/migrate.mjs` | Applique toutes les migrations Drizzle via `drizzle-orm/postgres-js/migrator` |
| `scripts/ensure-tables.mjs` | Bootstrap idempotent `CREATE TABLE IF NOT EXISTS` — s'exécute au démarrage du conteneur **en plus des** migrations |
| `scripts/migrate-concurrent-indexes.mjs` | Échanges d'index non transactionnels (ex. index locale des connections) |

**FACT:** La migration `0008_semantic_and_morphology.sql` active pgvector et crée `verse_embeddings` + `word_morphology` :

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE verse_embeddings (ref text PRIMARY KEY, embedding vector(768) NOT NULL, ...);
CREATE INDEX verse_embeddings_hnsw_idx ON verse_embeddings USING hnsw (embedding vector_cosine_ops);
```

**FACT:** Les tests d'intégration lancent `pgvector/pgvector:pg16` via Testcontainers, exécutent `migrate()`, puis appliquent manuellement l'index concurrent de la migration 0020 (`__tests__/integration/global-setup.ts`).

**À COMPRENDRE MAINTENANT :** Les changements de schéma exigent une **migration réversible** validée par `bun run test:integration` (AGENTS.md). Ne modifiez pas la base de production à la main sans fichier de migration.

---

## Le corpus local du Coran (`verses`)

**FACT:** La clé primaire est la chaîne canonique `"surah:ayah"` (`ref`).

| Colonne | Source |
| --- | --- |
| `arabicText` | alquran.cloud `quran-uthmani` |
| `translation` | alquran.cloud `en.sahih` (Saheeh International) |
| `surah`, `ayah` | Entiers dénormalisés pour le tri |

**FACT:** Chargé par `scripts/seed-quran.mjs` — attend **6236** ayahs ; upserts par lots.

**FACT:** `lib/quran/quran-corpus.ts` lit `verses` (+ `verse_translations` optionnel pour d'autres éditions). C'est la **couche d'hydratation de validation** pour les sorties IA et l'affichage.

**FACT:** L'affichage multilingue utilise `verse_translations` (migration `0018`) — chargé séparément via `scripts/seed-translations.mjs`. La recherche/l'affichage peut montrer une édition locale tandis que les embeddings restent indexés en anglais (ci-dessous).

---

## Morphologie lexicale (`word_morphology`)

**FACT:** Une ligne par **mot porteur de racine** par verset : `(ref, position, surface, root, lemma)`.

**FACT:** Chargé depuis `data/morphology/*.jsonl` versionné via `scripts/seed-morphology.mjs`.

**FACT:** Index :

| Index | Rôle |
| --- | --- |
| `(ref, position)` unique | Clé d'upsert |
| `(ref)` | Recherche des mots d'un verset |
| `(root)` | Trouver les versets partageant une racine |

**FACT:** `discoverCandidates(..., "root")` (`lib/ai/connection-discovery.ts`) :

1. Sélectionner les racines distinctes pour la `ref` source
2. Trouver d'autres refs partageant ces racines
3. Classer par `count(distinct root)` DESC
4. Respecter `excludeRefs`

**FACT:** Versets **sans lignes de morphologie** → liste de candidats vide → chemin IA legacy au premier expand (phase 7).

---

## Embeddings (`verse_embeddings`)

### Représentation

| Propriété | Valeur |
| --- | --- |
| Dimensions | **768** (`EMBEDDING_DIMENSIONS` dans `lib/ai/ai.ts`) |
| Modèle | `gemini-embedding-001` (override : `GEMINI_EMBEDDING_MODEL`) |
| Texte source | **`verses.translation`** (Saheeh anglais) |
| Dimensions natives du modèle | 3072 — réduites via `outputDimensionality: 768` |
| Index | **HNSW** avec `vector_cosine_ops` |

**FACT:** Les embeddings sont **toujours Gemini** — Anthropic n'a pas d'API d'embeddings (en-tête de `lib/ai/ai.ts`). C'est indépendant de `AI_PROVIDER` pour la génération de texte de connexion.

### Hors ligne : charger le corpus

**Script :** `scripts/embed-corpus.mjs`

```text
SELECT verses missing embedding for current model
→ batchEmbedContents (Gemini REST, batch size 100)
→ INSERT ... ON CONFLICT (ref) DO UPDATE
→ idempotent / resumable on 429
```

**FACT:** Nécessite `DATABASE_URL` + `GEMINI_API_KEY`. Relance sûre — ignore les refs déjà embeddées pour le modèle courant.

### En ligne : embedder les requêtes utilisateur

**Où :** `lib/quran/semantic-search.ts` → `embedQueryCached`

| Étape | Comportement |
| --- | --- |
| Normaliser la requête | `toLowerCase()` |
| Cache Redis | Clé `emb:q:<sha256>`, TTL 7 jours |
| Miss | `embed(query)` → même chemin Gemini REST que le corpus |
| Redis désactivé | Embed direct à chaque fois (repli `lib/infra/redis.ts`) |

**FACT:** Seules les **requêtes utilisateur** passent par embed live + cache Redis. Les vecteurs du corpus ne sont **jamais** recalculés à la requête.

**INFERENCE:** Embeddings indexés en anglais + `verse_translations` spécifiques à la locale signifie qu'une UI turque peut afficher du texte turc pour des voisins sémantiquement classés indexés en anglais — par conception (docstring de `searchByMeaning`).

---

## Requêtes pgvector — comment fonctionne la similarité

**Où :** `lib/quran/semantic-search.ts` → `nearest()`

```typescript
const similarity = sql<number>`1 - (${cosineDistance(verseEmbeddings.embedding, queryVec)})`;
return db
  .select({ ref: verseEmbeddings.ref, similarity })
  .from(verseEmbeddings)
  .where(excludeRefs ? notInArray(...) : undefined)
  .orderBy(desc(similarity))
  .limit(limit);
```

| Concept | Dans ce dépôt |
| --- | --- |
| Métrique de distance | **Distance cosinus** via `cosineDistance` de Drizzle |
| Score de similarité | `1 - cosineDistance` → plus élevé = plus proche |
| Index ANN | HNSW — plus proches voisins approximatifs à l'échelle |
| Exclusions | `notInArray(verseEmbeddings.ref, excludeRefs)` |

### Analogie Firestore (bref)

Si Firestore excelle dans les **recherches exactes par champ** (`where ref == "2:255"`), pgvector résout un autre problème : **« lequel des 6236 versets est le plus proche en sens de ce vecteur ? »** C'est une récupération par plus proches voisins, pas une requête d'égalité indexée.

---

## Surface API de la recherche sémantique

Trois fonctions exportées — même cœur `nearest()` :

| Fonction | Entrée | Sortie | Utilisée par |
| --- | --- | --- | --- |
| `searchByMeaning(query, limit, edition?)` | Texte libre | `SemanticMatch[]` | `GET /api/search` → `relatedByMeaning` |
| `similarVerses(ref, limit, excludeRefs?)` | Ref de verset | `SemanticMatch[]` | `GET /api/verse/.../similar` |
| `semanticCandidates(ref, limit, excludeRefs?)` | Ref de verset | `string[]` refs uniquement | `discoverCandidates` thématique/contrast |

**FACT:** `similarVerses` charge l'embedding source depuis la DB ; renvoie `[]` si aucun n'est stocké.

**FACT:** Les trois hydratent le texte d'affichage via `getVerses(refs, edition)` — classement depuis les vecteurs, **texte depuis le corpus local**.

### Route de recherche : mot-clé vs sémantique

**FACT:** `GET /api/search` exécute **deux chemins en parallèle** sur la page 1 :

| Chemin | Mécanisme | Persistance |
| --- | --- | --- |
| **Mot-clé** | API quran.com (`fetchQuranComSearch`) → refs → `hydrate()` depuis le corpus local | Index externe ; affichage depuis Postgres |
| **Liés par le sens** | `searchByMeaning` → pgvector | Embeddings dans Postgres ; embed de requête en live |

**FACT:** La section sémantique « liés » est **best-effort** — les échecs renvoient un `related` vide, invisible pour l'utilisateur (`relatedByMeaning` absorbe les erreurs).

**FACT:** Les deux chemins sont limités en débit sous le budget `consume(`search:${clientKey}`)`.

**FACT:** La table `search_log` enregistre les requêtes agrégées (mode `keyword` | `meaning`) quand le budget de logging le permet — pas d'attribution utilisateur.

---

## Ce qui est persisté vs calculé à l'exécution

| Donnée | Persistée ? | Où calculée |
| --- | --- | --- |
| Arabe du verset + traduction par défaut | **Oui** — chargée | `verses` |
| Traductions alternatives | **Oui** — chargées | `verse_translations` |
| Vecteurs d'embedding de versets | **Oui** — chargés | `verse_embeddings` |
| Racines lexicales | **Oui** — chargées | `word_morphology` |
| Arêtes de connexion IA | **Oui** — en cache miss | `connections` |
| Embedding de requête | **En cache** (Redis) ou éphémère | API Gemini |
| Résultats de recherche par mot-clé | **Non stockés** | quran.com par requête |
| Ordre de classement sémantique | **Calculé** à chaque requête | pgvector `ORDER BY similarity` |
| Disposition du canvas | **Client** localStorage / partage / workspace | Pas dans `connections` |

---

## Initialisation de la base (modèle mental dev local)

Ordre typique de **première** configuration :

```text
1. Postgres en cours d'exécution (docker-compose ou local) avec pgvector
2. node scripts/migrate.mjs          # appliquer les migrations Drizzle
3. node scripts/seed-quran.mjs       # verses (~6236 lignes)
4. node scripts/seed-morphology.mjs  # word_morphology (depuis data/morphology/)
5. node scripts/embed-corpus.mjs     # verse_embeddings (nécessite GEMINI_API_KEY)
6. Optionnel : seed-translations.mjs   # éditions non anglaises
```

**FACT:** Sans l'étape 5, la recherche sémantique et la découverte ancrée thématique/contrast renvoient vide — le repli IA legacy peut encore s'exécuter (phase 7).

**FACT:** Le panneau admin peut déclencher des jobs de seed via `lib/admin/job-runner.ts` → enregistre le résultat dans `job_runs`.

**UNKNOWN:** État exact du seed en production sur openhikmah.com — supposer un chargement complet en prod ; vérifier localement.

---

## Stratégie de tests d'intégration

**FACT:** `bun run test:integration` → Vitest avec :

| Paramètre | Valeur |
| --- | --- |
| Config | `vitest.integration.config.ts` |
| Setup global | Testcontainers `pgvector/pgvector:pg16` |
| Pool | `fileParallelism: false` — une DB partagée, fichiers sérialisés |
| Timeout | 30s test / 120s hook |

**FACT:** Les tests **mockent les API externes** (embed, Claude) mais utilisent **Postgres réel + pgvector réel** :

| Fichier | Prouve |
| --- | --- |
| `semantic-search.integration.test.ts` | Classement cosinus, excludeRefs, court-circuit requête vide |
| `connection-discovery.integration.test.ts` | Classement SQL racine sur de vraies lignes de morphologie |
| `graph.integration.test.ts` | Cache miss → persistance → cache hit ; limites de débit |

**FACT:** Le hook pre-push exécute les tests d'intégration (AGENTS.md) — **Docker requis** pour `git push`.

**FACT:** Les tests unitaires mockent `db` entièrement ; les tests d'intégration sont la couche de preuve pour le SQL et le comportement pgvector.

---

## Index importants (référence contributeur)

| Table | Index | Pourquoi |
| --- | --- | --- |
| `verses` | PK `ref` | Lookup verset O(1) |
| `connections` | UNIQUE `(from_ref, to_ref, kind, locale)` | Persistance IA idempotente |
| `connections` | `(from_ref, kind)` | Lecture cache pour une cellule |
| `verse_embeddings` | HNSW sur `embedding` | Requêtes ANN sémantiques |
| `word_morphology` | `(root)` | Jointure découverte racine |
| `bookmarks` | UNIQUE `(user_id, verse_ref)` | Un signet par ref par utilisateur |

---

## De bout en bout : requête de recherche sémantique

```mermaid
sequenceDiagram
  participant U as Utilisateur
  participant SD as SearchDialog
  participant API as GET /api/search
  participant KS as quran.com keyword
  participant SS as semantic-search.ts
  participant R as Redis
  participant G as Gemini embed API
  participant PG as Postgres pgvector
  participant QC as quran-corpus

  U->>SD: saisit "mercy and forgiveness"
  SD->>API: q=mercy...
  par Chemin mot-clé
    API->>KS: fetchQuranComSearch
    KS-->>API: refs + snippets
    API->>QC: getVerses(refs, edition)
  and Chemin sémantique (page 1)
    API->>SS: searchByMeaning(q)
    SS->>R: recherche cache embed
    alt cache miss
      SS->>G: embed(query)
      G-->>SS: vecteur 768 dim
      SS->>R: mettre en cache le vecteur
    end
    SS->>PG: ORDER BY 1 - cosine_distance LIMIT n
    PG-->>SS: refs + similarité
    SS->>QC: getVerses(refs, edition)
  end
  API-->>SD: results + optional related[]
  SD-->>U: résultats mot-clé + "liés par le sens"
```

---

## Récit lent : un classement sémantique

Vous saisissez **« patience in hardship »** dans la recherche.

La recherche par mot-clé interroge l'index de quran.com pour des correspondances littérales/tokens — ce chemin ne touche jamais pgvector. En parallèle, le serveur embedde votre phrase : Redis est consulté en premier ; en cas de miss, Gemini renvoie un vecteur 768 dimensions représentant le sens anglais de votre requête.

Ce vecteur est comparé aux **6236 vecteurs pré-stockés** — un par verset, chacun construit à partir de la traduction Saheeh anglaise lorsque `embed-corpus.mjs` a tourné. Postgres utilise l'index HNSW pour trouver les refs les plus proches sans scanner chaque ligne.

Les meilleures correspondances reviennent comme refs + scores de similarité. L'application hydrate ensuite l'arabe et la traduction depuis les tables locales `verses` / `verse_translations` — le texte affiché reste donc cohérent avec le corpus même si le classement était indexé en anglais.

Si les embeddings n'ont jamais été chargés, toute cette branche ne renvoie rien silencieusement ; les résultats par mot-clé fonctionnent toujours.

---

## À COMPRENDRE MAINTENANT

1. **`schema.ts` est le contrat des tables** — migrations + requêtes Drizzle en découlent.
2. **pgvector + HNSW + distance cosinus** alimentent tout le classement sémantique — pas le LLM.
3. **Les embeddings sont toujours Gemini, toujours 768 dim, toujours depuis le texte de traduction anglais** pour les lignes du corpus.
4. **`discoverCandidates` lit `word_morphology` ou `verse_embeddings`** — les scripts de seed comptent pour les connexions ancrées.
5. **`connections` est écrit en cache miss IA** — séparé du canvas et de la recherche par mot-clé.
6. **Les tests d'intégration avec Testcontainers** sont la preuve du dépôt pour SQL/pgvector — les tests unitaires mockent la DB.

---

## UTILE PLUS TARD

- `scripts/prewarm-graph.mjs` — préchauffage offline du graphe (ops admin).
- `scripts/backfill-connections.ts` — génération batch de connexions.
- `lib/infra/rate-limit.ts` — compteurs fenêtre fixe dans Postgres (table `rate_limits`).
- Chargement de `verse_translations` pour une UI non anglaise sans ré-embedder.
- Paramètres de tuning HNSW — non exposés dans le code applicatif aujourd'hui ; valeurs par défaut de Postgres/pgvector.

---

## IGNORER POUR L'INSTANT

- Tables social/challenge/amitié — sans lien avec le pipeline de recherche.
- Cache `name_content` / noms divins — autre motif de cache IA, même idée write-once.
- Internes Redis quand `REDIS_URL` n'est pas défini — le repli in-process suffit pour le dev local.
- Schéma exact de réponse API quran.com — traiter comme index de recherche externe uniquement.

---

## Questions de contrôle — Phase 9

1. Pourquoi les embeddings du corpus et les embeddings de requête doivent-ils utiliser le même modèle et la même dimensionalité ?
2. Quelle table `discoverCandidates("2:255", "root")` interroge-t-il, et que signifie un résultat vide ?
3. En quoi `searchByMeaning` diffère-t-il de la recherche par mot-clé en source de données et persistance ?
4. Qu'est-ce qui prouve que le classement pgvector fonctionne en CI ?
5. Nommez les trois scripts de seed qui peuplent les tables d'ancrage de niveau 1 (hors traductions).

---

**Suite :** [Phase 10 — Lancer l'application localement](./phase-10-run-locally.md) — configuration Bun, `.env.local`, quelles fonctionnalités nécessitent quelles clés, Docker, serveur de dev.

Dites **« continuer vers la Phase 10 »** quand vous êtes prêt, ou demandez un zoom sur un sujet base de données/recherche.

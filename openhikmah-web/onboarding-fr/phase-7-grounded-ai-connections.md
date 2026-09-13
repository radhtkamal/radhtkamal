# Phase 7 — Connexions IA ancrées (plongée profonde)

> **Prérequis :** Phases [1](./phase-1-what-is-openhikmah.md)–[6](./phase-6-runtime-walkthroughs.md)  
> **Tags d'évidence :** **FACT** (fait vérifiable dans le code) · **INFERENCE** (déduction raisonnée) · **UNKNOWN** (non documenté ou incertain)

La Phase 6 a tracé expand-to-canvas au niveau résumé. La Phase 7 est la **plongée dans le modèle de confiance** — le pipeline qui rend OpenHikmah « ancré » plutôt qu'un chatbot Coran libre.

**Phrase centrale (à mémoriser) :**

> **Les données découvrent ; l'IA articule.**

Le modèle peut **sélectionner** parmi de vrais versets candidats et **expliquer** pourquoi — il ne peut **pas** inventer de références de versets sur le chemin préféré.

---

## La séparation des pouvoirs

| Rôle | Module | Ce qu'il fait | Probabiliste ? |
| --- | --- | --- | --- |
| **Discover** | `lib/ai/connection-discovery.ts` | Retourne jusqu'à 12 refs `surah:ayah` réelles depuis Postgres | **Non** — SQL / pgvector |
| **Articulate** | `lib/ai/connection-generator.ts` → `generateGroundedConnections` | Le LLM choisit ≤3 parmi les candidats + écrit les raisons | **Oui** — mais refs contrôlées |
| **Orchestrate + cache** | `lib/ai/graph-service.ts` | Cache hit → skip IA ; miss → discover → generate → persist | Mixte |
| **Legacy fallback** | `generateConnections` | Le LLM propose des refs depuis la mémoire ; filtre `isValidRef` + `getVerses` | **Oui** — uniquement si données d'ancrage absentes |

**FACT :** L'en-tête de `connection-generator.ts` indique qu'il est *« The ONLY module that calls the AI »* pour les connexions.

**FACT :** En-tête de `graph-service.ts` : *« Reads connections from Postgres; only on a miss does it call the AI, then writes the result back so every later reader gets it for free. »*

---

## Pipeline de bout en bout (un diagramme)

```mermaid
flowchart TB
  subgraph client ["Navigateur"]
    VN[VerseNode.handleExpandSelect]
    PE[setPendingExpand]
    RE[HikmahCanvas.runExpansion]
    GER[getExpansionRefs]
    VN --> PE --> RE
    RE --> GER
  end

  subgraph api ["Frontière API"]
    RT[POST /api/connections]
    IV[isValidRef + plafonds champs]
    RT --> IV
  end

  subgraph graph ["lib/ai/graph-service.ts"]
    GC[getConnections]
    RA[readActiveRows — lecture cache]
    SF[dédupe singleFlight]
    GFC[generateConnectionsForCell]
    GLC[generateLocalizedCell — non-en]
    HY[hydrate → resolveVerse]
    GC --> RA
    RA -->|hit| HY
    RA -->|miss| SF
    SF --> GFC
    SF --> GLC
  end

  subgraph discover ["Les données découvrent"]
    DC[discoverCandidates]
    RC[rootCandidates — word_morphology]
    SC[semanticCandidates — verse_embeddings]
    DC --> RC
    DC --> SC
  end

  subgraph ai ["L'IA articule"]
    GGC[generateGroundedConnections]
    GC2[generateConnections — legacy]
    PAR[parseRawConnections]
    ALW["allowed.has(c.ref)"]
    GGC --> PAR --> ALW
  end

  subgraph persist ["Postgres"]
    CONN[(connections)]
    AIG[(ai_generations)]
  end

  RE -->|excludeRefs| RT
  RT --> GC
  GFC --> DC
  DC -->|candidates.length > 0| GGC
  DC -->|candidates vides + première requête| GC2
  DC -->|candidates vides + get more| EMPTY[return []]
  GGC --> CONN
  GC2 --> CONN
  GGC --> AIG
  GC2 --> AIG
  CONN --> HY
  HY --> RE
```

---

## Les douze questions du pipeline (répondues)

La spec d'onboarding originale les posait dans l'ordre. Utilisez ceci comme checklist ; les Étapes 0–9 ci-dessous développent chaque réponse.

| # | Question | Réponse (chemin ancré) |
| --- | --- | --- |
| 1 | Qu'est-ce qui démarre la requête ? | L'utilisateur sélectionne Theme/Root/Contrast → `VerseNode` → `setPendingExpand` → `HikmahCanvas.runExpansion` → `POST /api/connections` |
| 2 | Comment les versets candidats sont déterminés ? | `discoverCandidates(fromRef, kind, 12, excludeRefs)` — SQL sur `word_morphology` (root) ou voisins pgvector (thematic/contrast) |
| 3 | Quelles parties utilisent les racines arabes ? | **`kind === "root"` uniquement** → `rootCandidates()` dans `connection-discovery.ts` |
| 4 | Quelles parties utilisent les embeddings ? | **`thematic` et `contrast`** → `semanticCandidates()` → `similarVerses()` → `verse_embeddings` + distance cosinus |
| 5 | Où Claude/Gemini entre ? | Uniquement sur cache miss, dans `generateGroundedConnections` ou legacy `generateConnections` → `callAIDetailed(..., { feature: "connections" })` → `resolveProvider("connections")` choisit Claude (défaut) ou Gemini selon flags admin / env (`lib/ai/ai.ts`) |
| 6 | Quel contexte le modèle reçoit ? | Ref source + arabe + traduction + liste candidats numérotée (`- ref — translation`) + tâche de sélection par kind + cadrage Maturidi/Hanafi + Tanzih ajouté |
| 7 | Quel schéma/format est attendu ? | Tableau JSON uniquement : `[{ "ref": "surah:ayah", "reason": "one sentence" }, ...]` — au plus 3 entrées ; pas d'enveloppe markdown |
| 8 | Comment la sortie est parsée ? | `parseRawConnections(text)` — regex extrait le premier `[...]`, `JSON.parse`, filtre objets avec `ref` + `reason` chaînes non vides |
| 9 | Comment les références de versets sont vérifiées ? | **Ancré :** `allowed.has(c.ref)` où `allowed` = ensemble discovery. **Legacy :** `isValidRef` + lookup corpus `getVerses`. Les deux chemins excluent `fromRef`. |
| 10 | Que se passe-t-il si la validation échoue ? | Malformé/non parsable → `ConnectionParseError` → API **502** (retry). Parse valide mais aucune ref survivante → `[]` (200). Rate limit → **429**. |
| 11 | Quelles contraintes théologiques sont ajoutées ? | `tanzihDirective()` ajoute `TANZIH_CONSTRAINT` ; le prompt scholar cadre la tradition Maturidi/Hanafi — aucun n'est supprimable via overrides prompt admin |
| 12 | Qu'est-ce qui devient une arête canvas ? | `ConnectionResult[]` retourné → `runExpansion` → `addVerseNode` + `buildConnectionEdge` + `addConnectionEdge` — l'arête porte `kind` + `reason` dans `edge.data` |

**FACT :** Les embeddings pour la discovery utilisent Gemini (`embed()` dans `semantic-search.ts`) ; la **génération de texte** de connexion utilise Claude ou Gemini via `callAIDetailed`. Ce sont des sites d'appel séparés.

**UNKNOWN :** Quel provider tourne en production sans inspecter env/flags admin déployés — le défaut code est Claude si non défini.

---

## Récit lent (lire une fois, puis tracer dans le code)

Imaginez que vous expandez **2:255** par **Theme**.

Vous avez déjà le verset sur le canvas — son arabe et sa traduction voyagent avec la requête. Le client envoie aussi chaque ref cible thématique déjà connectée depuis ce nœud (`excludeRefs`), pour que « get more » ne puisse pas répéter les mêmes trois ayahs.

L'API valide la forme de la ref et délègue à `getConnections`. Postgres est consulté d'abord : si cette cellule `(2:255, thematic, locale)` a été générée avant et que ces cibles ne sont pas exclues, vous obtenez des lignes en cache — **pas de facture LLM**.

Sur un miss, le serveur découvre jusqu'à douze voisins réels par similarité d'embedding — math déterministe, pas mémoire du modèle. Ces refs s'hydratent depuis le corpus Coran local. Seulement alors le modèle s'exécute : il voit le verset source, la liste candidats, et les instructions de choisir les trois meilleurs liens thématiques et d'expliquer chacun en une phrase — dans le cadrage Maturidi/Hanafi et Tanzih.

Le JSON du modèle revient. Le code extrait le tableau, supprime toute ref hors des douze, supprime les auto-liens, plafonne à trois, et persiste les survivants dans `connections`. La réponse hydrate les cartes versets complètes plus les raisons. De retour dans le navigateur, les nœuds s'épanouissent radialement ; les arêtes apparaissent avec des pastilles d'explication cliquables dans la sidebar.

Si le modèle refuse ou retourne du JSON invalide, vous obtenez un 502 — pas un canvas vide silencieux. Si le pool ancré est vraiment épuisé sur « get more », vous obtenez un tableau vide et une notice informative « no more connections ». Cette distinction protège à la fois la théologie et l'UX.

---

## Étape 0 — Ce qui déclenche le pipeline

**FACT :** L'utilisateur choisit Theme / Root / Contrast dans `ExpandMenu` → `VerseNode.handleExpandSelect` → `setPendingExpand({ nodeId, ref, kind })`.

**FACT :** `HikmahCanvas` `useEffect` sur `pendingExpand` efface l'état pending et appelle `runExpansion` avec `arabicText`, `translation` et position du nœud source.

**FACT :** Le premier nœud canvas auto-expand **thematic** via `pendingAutoExpand` — même chemin `runExpansion`.

---

## Étape 1 — Le client construit `excludeRefs` (« get more »)

| | |
| --- | --- |
| **Where** | `store/canvas.ts` → `getExpansionRefs(nodeId, kind)` |
| **Algorithm** | Arêtes où `e.source === nodeId && e.data.kind === kind` → refs nœuds cibles |
| **Why** | Un expand répété doit faire surface à de **nouvelles** connexions, pas re-servir les mêmes cibles |

**FACT :** `runExpansion` envoie `{ fromRef, kind, arabicText, translation, excludeRefs }` à `POST /api/connections`.

**INFERENCE :** `excludeRefs` est une **liste de session côté client** (ce qui est déjà sur le canvas pour ce kind), pas l'historique complet du graphe Postgres. Un verset connecté sur le canvas est exclu du prochain « get more » même s'il venait d'une session d'expansion différente.

---

## Étape 2 — Frontière API (validation frontière système)

**Where:** `app/api/connections/route.ts`

| Vérification | Objectif |
| --- | --- |
| Champs requis | `fromRef`, `kind`, `arabicText`, `translation` |
| `kind ∈ { thematic, root, contrast }` | Rejette les edge kinds inconnus |
| `isValidRef(fromRef)` | `"surah:ayah"` canonique dans les bornes Hafs réelles |
| Tableau `excludeRefs`, chaque `isValidRef`, max 100 | Empêche l'injection prompt non bornée |
| Longueur texte ≤ 5000 | Borne la taille du prompt IA |

**FACT :** En cas de succès, appelle `getConnections(..., { clientKey, excludeRefs, locale })` où `locale = await getUiLocale()`.

**FACT :** Mapping erreurs :

| Erreur | HTTP | Comportement client |
| --- | --- | --- |
| `RateLimitError` | 429 | Notice `connectionsFailed` |
| `ConnectionParseError` | 502 | Mérite retry — **pas** « no connections » |
| `[]` vide | 200 | Le client distingue épuisé vs vide première fois |

**À COMPRENDRE MAINTENANT :** Un tableau JSON vide et un échec parse 502 ont des **sémantiques différentes**. Vide = « rien à montrer » bien formé. 502 = échec upstream transitoire — ne doit **pas** être mis en cache comme épuisement.

---

## Étape 3 — `getConnections` : orchestration cache-first

**Where:** `lib/ai/graph-service.ts` → `getConnections`

### 3a — Lecture cache

**FACT :** `readActiveRows(fromRef, kind, locale, excludeRefs)` sélectionne depuis `connections` où :

- `from_ref`, `kind`, `locale` correspondent
- `status = 'active'`
- `to_ref NOT IN excludeRefs` (si fourni)
- ordonné par `toRef`, limite 200

**FACT :** Pour les locales non-`en`, un cache hit exige que **chaque** ligne anglaise active ait une ligne locale correspondante (porte de complétude ref par ref). Des ensembles locale partiels ré-entrent le chemin miss.

**FACT :** Sur hit → `hydrate(rows, kind)` → `resolveVerse(toRef)` pour chaque ligne → `ConnectionResult[]`. **Pas d'IA.**

### 3b — Gardes cache miss

| Garde | Objectif |
| --- | --- |
| `consume(`gen:${clientKey}`)` | Rate-limit le chemin miss coûteux par IP client |
| `singleFlight(key, factory)` | Les misses identiques concurrentes partagent une génération |
| `resolveProvider` / `resolveModel` | Provider+model résolus une fois, propagés dans key + persist |

**FACT :** `cellKey` = `` `${fromRef}:${kind}:${locale}:${provider}:${model}:${sorted excludeRefs}` ``

**FACT :** Miss non-`en` → `generateLocalizedCell` : assure que les lignes anglaises existent, puis `translateReason` par ligne manquante (consommation budget séparée). La **sélection** anglaise est toujours canonique ; la locale affecte uniquement le **texte de raison**.

---

## Étape 4 — `generateConnectionsForCell` : la bifurcation

**Where:** `graph-service.ts` → `generateConnectionsForCell` (exporté aussi pour backfill admin)

```typescript
const candidates = await discoverCandidates(fromRef, kind, undefined, excludeRefs);
const calledAI = candidates.length > 0 || excludeRefs.length === 0;

const generated =
  candidates.length > 0
    ? await generateGroundedConnections(...)
    : excludeRefs.length > 0
      ? []
      : await generateConnections(...);  // legacy
```

### Table de décision

| `candidates` | `excludeRefs` | Chemin | IA appelée ? | Signification typique |
| --- | --- | --- | --- | --- |
| > 0 | any | **Ancré** | Oui | Cas normal |
| 0 | > 0 | **Return []** | Non (`calledAI` toujours true*) | Pool ancré « get more » épuisé |
| 0 | 0 | **Legacy** | Oui | Morphologie/embeddings non seedés pour le verset |

\* **FACT :** `calledAI = candidates.length > 0 || excludeRefs.length === 0` — pool vide sur « get more » marque toujours `calledAI: true` pour que le backfill admin enregistre l'épuisement correctement.

**À COMPRENDRE MAINTENANT :** Le chemin legacy **ne s'exécute jamais** quand `excludeRefs.length > 0`. Sans cette règle, « get more » régénérerait des refs similaires depuis la mémoire du modèle et annulerait la pagination.

---

## Étape 5 — `discoverCandidates` : discovery déterministe

**Where:** `lib/ai/connection-discovery.ts`

**FACT :** Limite par défaut = **12** candidats par passe de génération.

### Par kind

| Kind | Table source | Algorithme |
| --- | --- | --- |
| `root` | `word_morphology` | Racines distinctes pour la ref source → versets partageant ces racines → classer par `count(distinct root)` DESC |
| `thematic` | `verse_embeddings` | `semanticCandidates` → plus proches voisins pgvector |
| `contrast` | `verse_embeddings` | **Même pool voisins que thematic** — l'IA sélectionne les véritablement opposés |

**FACT :** `semanticCandidates` (`lib/quran/semantic-search.ts`) : charge l'embedding source → `nearest()` similarité cosinus → retourne refs. Retourne `[]` si la source n'a **pas** d'embedding stocké.

**FACT :** `excludeRefs` propagé au SQL (`notInArray`) pour les chemins root et semantic.

**FACT :** `[]` vide depuis discovery signifie *« pas de données d'ancrage pour ce verset »* — **pas** une erreur. Les appelants décident fallback vs épuisement.

### Dépendances de seeding (offline)

| Données | Script | Nécessaire pour |
| --- | --- | --- |
| `verses` | `scripts/seed-quran.mjs` | Toute hydratation |
| `word_morphology` | `scripts/seed-morphology.mjs` | Discovery `root` |
| `verse_embeddings` | `scripts/embed-corpus.mjs` | Discovery `thematic` / `contrast` |

**INFERENCE :** Une DB locale dev fraîche avec Coran seedé mais **sans** embeddings/morphologie touchera **legacy** `generateConnections` au premier expand — jusqu'à ce que les tables d'ancrage soient peuplées.

---

## Étape 6 — `generateGroundedConnections` : l'IA articule (chemin préféré)

**Where:** `lib/ai/connection-generator.ts`

### 6a — Hydrater les candidats avant le prompt

```typescript
const verseMap = await getVerses(candidateRefs);
const candidates = candidateRefs
  .map((ref) => verseMap.get(ref))
  .filter((v) => v !== undefined && v.ref !== fromRef);
if (candidates.length === 0) return [];  // no LLM call
```

**FACT :** Si aucun candidat ne se résout depuis le corpus local, retourne `[]` **sans appeler l'IA**.

### 6b — Construction du prompt

| Pièce | Source |
| --- | --- |
| Template de base | `getPrompt("connection.selection", SELECTION_FALLBACK_TEMPLATE)` |
| Formulation tâche | `KIND_SELECTION[kind]` — diffère par kind |
| Liste candidats | `- {ref} — {translation}` par verset |
| Cadrage Maturidi/Hanafi | Dans le template |
| Règle : choisir UNIQUEMENT dans la liste | Dans le template |
| **Tanzih** | `tanzihDirective()` — **ajouté, non surchargeable** |
| Locale | `languageDirective(locale)` — ajouté pour non-`en` |

**FACT :** `tanzihDirective()` ajoute `TANZIH_CONSTRAINT` depuis `lib/ai/theological-constraints.ts` :

> *« strict Tanzih (divine transcendence): never describe or imply physical form, spatial location, or resemblance to created things (Tashbih) »*

**FACT :** Les directives Tanzih et langue ne sont **pas** des `{{placeholders}}` dans les templates surchargeables admin — elles sont ajoutées après `renderTemplate` pour que les overrides prompt DB ne puissent pas les omettre (voir Phase 3).

### 6c — Appel LLM + journal audit

**FACT :** `callAIDetailed(prompt, { feature: "connections", provider, model, ... })`

**FACT :** Insert best-effort dans `ai_generations` (fromRef, kind, model, tokens, promptVersion). L'échec de logging ne fait **pas** échouer la génération.

### 6d — Parse + validation (la porte anti-hallucination)

```typescript
const allowed = new Set<string>(candidates.map((v) => v.ref));
const chosen = parseRawConnections(text)
  .filter((c) => allowed.has(c.ref) && c.ref !== fromRef)
  .slice(0, 3);
```

| Étape | Règle déterministe |
| --- | --- |
| `parseRawConnections` | Extraire le premier tableau JSON `[...]` ; valider la forme ; lancer `ConnectionParseError` si malformé |
| `allowed.has(c.ref)` | **Supprimer toute ref hors ensemble discovery** — même si le modèle « connaît » un verset valide |
| `c.ref !== fromRef` | Ne jamais connecter un verset à lui-même |
| `.slice(0, 3)` | Plafonner à 3 arêtes par génération |
| Mapper via `verseMap` | Hydrater `ConnectionResult` complet |

**FACT :** Le test prouve que le modèle retournant `9:99` quand absent de la liste candidats → supprimé (`connection-generator.test.ts` : *« rejects any ref the model returns that was not in the candidate set »*).

**FACT :** `[]` bien formé du modèle → sélection vide valide (pas de throw). Prose malformée / refus → `ConnectionParseError` → API 502.

**À COMPRENDRE MAINTENANT :** La validation ancrée est **`allowed.has(c.ref)`**, pas seulement `isValidRef`. Une ref fabriquée mais syntaxiquement valide est rejetée si la discovery ne l'a pas surfacée.

---

## Étape 7 — `generateConnections` : fallback legacy

**When:** `discoverCandidates` retourne `[]` **et** `excludeRefs.length === 0` (miss première fois, pas de données d'ancrage).

**FACT :** Utilise `getPrompt("connection.legacy", LEGACY_FALLBACK_TEMPLATE)` + `KIND_INSTRUCTIONS[kind]`.

**FACT :** Le modèle peut proposer n'importe quelles refs — filtrées par :

1. `parseRawConnections`
2. `isValidRef(c.ref) && c.ref !== fromRef`
3. `getVerses(candidates.map(c => c.ref))` — **doit exister dans le corpus local**
4. `.slice(0, 3)`

**FACT :** Les refs qui passent `isValidRef` mais manquent le lookup corpus sont silencieusement supprimées — pas d'hydratation API externe sur ce chemin.

**INFERENCE :** Legacy est un **chemin bootstrap** pour données d'ancrage non seedées, pas le comportement produit long terme. La production avec seeds complets devrait presque toujours toucher le chemin ancré.

---

## Étape 8 — Persistance et gestion des courses

**Where:** fin de `generateConnectionsForCell`

**FACT :** Insert dans `connections` :

```typescript
{ fromRef, toRef: g.ref, kind, reason: g.reason, model, locale: "en" }
.onConflictDoNothing()
```

**FACT :** Index unique sur `(from_ref, to_ref, kind, locale)` — générations concurrentes sur deux instances : le perdant relit la raison persistée du gagnant.

**FACT :** L'échec de persist est loggé + `incr("gen_persist_failed")` — l'appelant retourne toujours les résultats générés **cette requête**, mais la prochaine requête paie à nouveau le coût IA.

**FACT :** `connections.status` peut être `active | flagged | retired` — file de revue admin à `/admin/connections`. Le canvas lit les lignes **active** uniquement.

---

## Étape 9 — Réponse retour au canvas

**FACT :** `hydrate` / `ConnectionResult[]` mappé inclut : ref, arabicText, translation, noms sourates, **reason**, **kind**.

**FACT :** Boucle `runExpansion` :

- Tableau vide + `excludeRefs.length > 0` → `ExhaustedExpansionError` → « no more connections »
- Tableau vide + première requête → « no connections found » générique
- Non vide → placement nœuds échelonné (`radialPos`, `findFreeSlot`), `addVerseNode`, `addConnectionEdge`

**FACT :** Si la ref cible est déjà sur le canvas → dessiner l'arête uniquement (pas de nœud dupliqué).

---

## Trois edge kinds — même pipeline, discovery et intention de sélection différentes

| Kind | Question discovery | Question sélection IA (`KIND_SELECTION`) |
| --- | --- | --- |
| **thematic** | Qui est sémantiquement proche ? (embeddings) | Choisir 3 qui **partagent le thème théologique** |
| **root** | Qui partage les racines arabes ? (morphologie) | Choisir 3 où la racine partagée **porte un sens pertinent** |
| **contrast** | Qui est sémantiquement proche ? (même pool que thematic) | Choisir 3 avec le **concept opposé le plus clair** |

**INFERENCE :** Contrast est le kind le plus subtil — la discovery est **basée sur similarité**, l'articulation sur **l'opposition**. Le modèle doit faire le travail théologique dans la sélection, pas la discovery.

---

## Déterministe vs probabiliste — checklist contributeur

| Étape | Déterministe ? | Peut-on « corriger un test » en assouplissant ? |
| --- | --- | --- |
| `isValidRef` | Oui | **Jamais** |
| SQL `discoverCandidates` | Oui | Corriger les données seed, pas les bornes |
| `allowed.has(c.ref)` | Oui | **Jamais** |
| Hydratation `getVerses` | Oui | Corriger le corpus, pas sauter le lookup |
| `parseRawConnections` | Oui | Corriger prompt/modèle, pas le parser |
| Append `tanzihDirective` | Oui | **Jamais** supprimer |
| Formulation raison | Non (LLM) | Revue admin + changements prompt, avec disclosure PR |
| Quels 3 des 12 candidats | Non (LLM) | Accepter la variance dans la validation |

---

## Modes d'échec que vous verrez en développement

| Symptôme | Cause probable | Où regarder |
| --- | --- | --- |
| Premier expand → vide, pas d'erreur | Pas d'embeddings/morphologie seedés ; legacy n'a rien retourné de valide | Exécuter scripts seed ; vérifier retour `discoverCandidates` |
| « Get more » → « no more connections » | Pool ancré épuisé pour le kind | Attendu — `excludeRefs` + candidats vides |
| Expand → toast erreur, 502 dans le réseau | `ConnectionParseError` — refus/JSON tronqué | Réponse modèle ; retry |
| 429 sur expand | Rate limit sur chemin miss | `lib/infra/rate-limit.ts` ; cache hits non affectés |
| Mêmes connexions à chaque fois | Cache hit — fonctionnement prévu | Table `connections` |
| UI turque, raisons anglaises brièvement | Traduction locale en cours / budget épuisé | `generateLocalizedCell` serve partiel |

---

## Tests clés (lire avant de modifier le pipeline)

| Fichier | Ce qu'il verrouille |
| --- | --- |
| `__tests__/lib/ai/connection-generator.test.ts` | Erreurs parse, `allowed.has`, hydratation corpus, Tanzih/locale dans prompts |
| `__tests__/lib/ai/connection-discovery.test.ts` | Classement root, excludeRefs en SQL, routage kind |
| `__tests__/lib/ai/graph-service.test.ts` | Cache hit/miss, single-flight, excludeRefs, complétude locale |
| `__tests__/integration/graph.integration.test.ts` | Bout en bout avec Postgres réel |
| `__tests__/components/canvas/HikmahCanvas.test.tsx` | Comportement client vide vs épuisé |

---

## Fichiers à garder ouverts en debug expand

| Fichier | Rôle |
| --- | --- |
| `components/canvas/HikmahCanvas.tsx` | `runExpansion`, sémantique erreur client |
| `store/canvas.ts` | `getExpansionRefs` |
| `app/api/connections/route.ts` | Frontière HTTP + mapping erreurs |
| `lib/ai/graph-service.ts` | Cache, single-flight, locale, persist |
| `lib/ai/connection-discovery.ts` | SQL candidats |
| `lib/quran/semantic-search.ts` | Voisins embedding |
| `lib/ai/connection-generator.ts` | Prompts, parse, ancré vs legacy |
| `lib/quran/quran-corpus.ts` | `isValidRef`, `getVerses` |
| `lib/ai/theological-constraints.ts` | `TANZIH_CONSTRAINT` |
| `lib/infra/db/schema.ts` | `connections`, `ai_generations`, index |

---

## À COMPRENDRE MAINTENANT

1. **Chemin ancré :** discovery produit refs → IA sélectionne sous-ensemble → `allowed.has(c.ref)` impose le sous-ensemble.
2. **Chemin legacy :** uniquement quand discovery vide **et** pas une requête « get more ».
3. **`[]` vide ≠ échec parse** — statut HTTP différent, UX client différente, sémantique backfill admin différente.
4. **La sélection anglaise est canonique** — non-`en` traduit les raisons, ne re-choisit pas les versets.
5. **Postgres `connections` est le graphe produit** — le canvas est layout de session ; le partage sauvegarde JSON canvas, pas cette table.
6. **Ne jamais assouplir la validation verset pour passer un test** — corriger données, seeds ou prompts à la place.

---

## UTILE PLUS TARD

- Boucle backfill admin : `lib/ai/connection-batch.ts`, `connection_coverage.exhaustedAt`, rotation clés Gemini (`AGENTS.md`).
- Overrides prompt admin : `lib/ai/prompt-registry.ts` — peut changer la formulation, pas l'append Tanzih.
- Sélection provider : `lib/ai/ai.ts` → `resolveProvider("connections")`.
- Setup test d'intégration avec Testcontainers pour vérification complète chemin miss.

---

## IGNORER POUR L'INSTANT

- Dimensions exactes modèle embedding et classification quota Gemini (`lib/ai/gemini-errors.ts`) — territoire Phase 9.
- Workflow UI revue connexions admin — sauf si vous contribuez à l'outillage modération.
- Comptes dénormalisés `connection_coverage` — comptabilité backfill uniquement.

---

## Questions checkpoint Phase 7

Répondez de mémoire, puis vérifiez dans le code :

1. Quelle fonction impose que le modèle ne peut pas introduire une ref hors de l'ensemble discovery ?
2. Sous quelles deux conditions `generateConnections` (legacy) s'exécute ?
3. Pourquoi contrast utilise `semanticCandidates` au lieu d'une requête SQL séparée ?
4. Que se passe-t-il si le modèle retourne `[]` bien formé vs un refus en prose ?
5. Où Tanzih est injecté, et pourquoi ce n'est pas un `{{placeholder}}` ?

---

**Suivant :** [Phase 8 — Modèle d'état et canvas (détaillé)](./phase-8-state-and-canvas.md) — stores Zustand, persistance, sérialisation share, et comment l'état canvas se rapporte à (mais n'est pas) le graphe Postgres.

Dites **« continue to Phase 8 »** quand vous êtes prêt, ou demandez un zoom sur n'importe quelle étape ci-dessus.

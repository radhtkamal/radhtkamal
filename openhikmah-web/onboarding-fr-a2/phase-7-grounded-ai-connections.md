# Phase 7 — Connections AI grounded (plongée profonde)

> **Prérequis :** Phases [1](./phase-1-what-is-openhikmah.md)–[6](./phase-6-runtime-walkthroughs.md)  
> **Tags d'évidence :** **FACT** · **INFERENCE** · **UNKNOWN**

La Phase 6 a tracé expand-to-canvas en résumé. La Phase 7 est la **plongée modèle de confiance** — le pipeline qui rend OpenHikmah « grounded » (ancré) au lieu d'un chatbot Coran libre.

**Phrase centrale (à mémoriser) :**

> **Les données découvrent ; l'AI articule.**

Le modèle peut **choisir** parmi de vrais versets candidats et **expliquer** pourquoi — il ne peut **pas** inventer des références de versets sur le chemin préféré.

---

## La séparation des pouvoirs

| Rôle | Module | Ce qu'il fait | Probabiliste ? |
| --- | --- | --- | --- |
| **Discover** (découvrir) | `lib/ai/connection-discovery.ts` | Retourne jusqu'à 12 refs `surah:ayah` réelles depuis Postgres | **Non** — SQL / pgvector |
| **Articulate** (articuler) | `lib/ai/connection-generator.ts` → `generateGroundedConnections` | LLM (grand modèle de langage) choisit ≤3 parmi candidats + écrit raisons | **Oui** — mais refs contrôlées |
| **Orchestrate + cache** (orchestrer + cache) | `lib/ai/graph-service.ts` | Cache hit → skip AI ; miss → discover → generate → persist | Mixte |
| **Legacy fallback** (repli ancien) | `generateConnections` | LLM propose refs depuis mémoire ; `isValidRef` + `getVerses` filtrent | **Oui** — seulement si données grounding absentes |

**FACT :** L'en-tête de `connection-generator.ts` dit qu'il est *« The ONLY module that calls the AI »* pour les connections.

**FACT :** En-tête `graph-service.ts` : *« Reads connections from Postgres; only on a miss does it call the AI, then writes the result back so every later reader gets it for free. »*

---

## Pipeline de bout en bout (un diagramme)

```mermaid
flowchart TB
  subgraph client ["Browser"]
    VN[VerseNode.handleExpandSelect]
    PE[setPendingExpand]
    RE[HikmahCanvas.runExpansion]
    GER[getExpansionRefs]
    VN --> PE --> RE
    RE --> GER
  end

  subgraph api ["API boundary"]
    RT[POST /api/connections]
    IV[isValidRef + field caps]
    RT --> IV
  end

  subgraph graph ["lib/ai/graph-service.ts"]
    GC[getConnections]
    RA[readActiveRows — cache read]
    SF[singleFlight dedupe]
    GFC[generateConnectionsForCell]
    GLC[generateLocalizedCell — non-en]
    HY[hydrate → resolveVerse]
    GC --> RA
    RA -->|hit| HY
    RA -->|miss| SF
    SF --> GFC
    SF --> GLC
  end

  subgraph discover ["Data discovers"]
    DC[discoverCandidates]
    RC[rootCandidates — word_morphology]
    SC[semanticCandidates — verse_embeddings]
    DC --> RC
    DC --> SC
  end

  subgraph ai ["AI articulates"]
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
  DC -->|candidates empty + first request| GC2
  DC -->|candidates empty + get more| EMPTY[return []]
  GGC --> CONN
  GC2 --> CONN
  GGC --> AIG
  GC2 --> AIG
  CONN --> HY
  HY --> RE
```

---

## Les douze questions pipeline (répondues)

La spec onboarding originale les a posées dans l'ordre. Utilisez ceci comme checklist ; les Étapes 0–9 ci-dessous développent chaque réponse.

| # | Question | Réponse (chemin grounded) |
| --- | --- | --- |
| 1 | Qu'est-ce qui démarre la requête ? | L'utilisateur choisit Theme/Root/Contrast → `VerseNode` → `setPendingExpand` → `HikmahCanvas.runExpansion` → `POST /api/connections` |
| 2 | Comment les versets candidats sont déterminés ? | `discoverCandidates(fromRef, kind, 12, excludeRefs)` — SQL sur `word_morphology` (root) ou voisins pgvector (thematic/contrast) |
| 3 | Quelles parties utilisent racines arabes ? | **`kind === "root"` seulement** → `rootCandidates()` dans `connection-discovery.ts` |
| 4 | Quelles parties utilisent embeddings ? | **`thematic` et `contrast`** → `semanticCandidates()` → `verse_embeddings` + distance cosinus |
| 5 | Où Claude/Gemini entre ? | Seulement sur cache miss, dans `generateGroundedConnections` ou legacy `generateConnections` → `callAIDetailed(..., { feature: "connections" })` → `resolveProvider("connections")` choisit Claude (défaut) ou Gemini selon flags admin / env (`lib/ai/ai.ts`) |
| 6 | Quel contexte le modèle reçoit ? | Ref source + arabe + traduction + liste candidats numérotée (`- ref — translation`) + tâche sélection par kind + cadre Maturidi/Hanafi + Tanzih ajouté |
| 7 | Quel schéma/format est attendu ? | Tableau JSON seulement : `[{ "ref": "surah:ayah", "reason": "one sentence" }, ...]` — max 3 entrées ; pas de wrapper markdown |
| 8 | Comment la sortie est parsée ? | `parseRawConnections(text)` — regex extrait premier `[...]`, `JSON.parse`, filtre objets avec `ref` + `reason` chaînes non-vides |
| 9 | Comment les refs versets sont vérifiées ? | **Grounded :** `allowed.has(c.ref)` où `allowed` = ensemble discovery. **Legacy :** `isValidRef` + lookup corpus `getVerses`. Les deux chemins drop `fromRef`. |
| 10 | Que se passe-t-il si validation échoue ? | Malformé/non parsable → `ConnectionParseError` → API **502** (retry). Parse valide mais aucune ref survivante → `[]` (200). Rate limit → **429**. |
| 11 | Quelles contraintes théologiques sont ajoutées ? | `tanzihDirective()` ajoute `TANZIH_CONSTRAINT` ; prompt scholar cadre tradition Maturidi/Hanafi — aucun n'est supprimable via overrides prompt admin |
| 12 | Qu'est-ce qui devient une arête canvas ? | `ConnectionResult[]` retourné → `runExpansion` → `addVerseNode` + `buildConnectionEdge` + `addConnectionEdge` — arête porte `kind` + `reason` dans `edge.data` |

**FACT :** Les embeddings pour discovery utilisent Gemini (`embed()` dans `semantic-search.ts`) ; la **génération texte** connection utilise Claude ou Gemini via `callAIDetailed`. Ce sont des call sites séparés.

**UNKNOWN :** Quel provider tourne en production sans inspecter env/flags admin déployés — défaut code est Claude si non défini.

---

## Récit lent (lisez une fois, puis tracez dans le code)

Imaginez que vous expandez **2:255** par **Theme**.

Vous avez déjà le verset sur le canvas — son arabe et sa traduction voyagent avec la requête. Le client envoie aussi chaque ref cible thématique déjà connectée depuis ce nœud (`excludeRefs`), donc « get more » ne peut pas répéter les mêmes trois ayahs.

L'API valide la forme ref et passe à `getConnections`. Postgres est consulté d'abord : si cette cellule `(2:255, thematic, locale)` a été générée avant et ces cibles ne sont pas exclues, vous obtenez des lignes en cache — **pas de facture LLM**.

Sur un miss, le serveur découvre jusqu'à douze vrais voisins depuis similarité embedding — math déterministe, pas mémoire modèle. Ces refs s'hydratent depuis le corpus Coran local. Seulement alors le modèle tourne : il voit le verset source, la liste candidats, et les instructions pour choisir les trois meilleurs liens thématiques et expliquer chacun en une phrase — dans le cadre Maturidi/Hanafi et Tanzih.

Le JSON du modèle revient. Le code extrait le tableau, drop toute ref pas dans les douze, drop auto-liens, cap à trois, et persiste les survivants dans `connections`. La réponse hydrate cartes verset complètes plus raisons. Dans le navigateur, les nœuds s'épanouissent radialement ; les arêtes apparaissent avec pills explication cliquables dans la sidebar.

Si le modèle refuse ou retourne JSON garbage, vous obtenez 502 — pas un canvas vide silencieux. Si le pool grounded est vraiment épuisé sur « get more », vous obtenez un tableau vide et une notice « no more connections ». Cette distinction protège théologie et UX.

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
| **Why** | Répéter expand doit montrer des connections **nouvelles**, pas re-servir les mêmes cibles |

**FACT :** `runExpansion` envoie `{ fromRef, kind, arabicText, translation, excludeRefs }` à `POST /api/connections`.

**INFERENCE :** `excludeRefs` est une **liste session client** (ce qui est déjà sur le canvas pour ce kind), pas l'historique graphe Postgres complet. Un verset connecté sur le canvas est exclu du prochain « get more » même s'il vient d'une session expansion différente.

---

## Étape 2 — Frontière API (validation frontière système)

**Where :** `app/api/connections/route.ts`

| Check | Purpose |
| --- | --- |
| Required fields | `fromRef`, `kind`, `arabicText`, `translation` |
| `kind ∈ { thematic, root, contrast }` | Rejette kinds arête inconnus |
| `isValidRef(fromRef)` | `"surah:ayah"` canonique dans bornes Hafs réelles |
| `excludeRefs` array, each `isValidRef`, max 100 | Empêche injection prompt non bornée |
| Text length ≤ 5000 | Borne taille prompt AI |

**FACT :** Au succès, appelle `getConnections(..., { clientKey, excludeRefs, locale })` où `locale = await getUiLocale()`.

**FACT :** Mapping erreur :

| Error | HTTP | Client behavior |
| --- | --- | --- |
| `RateLimitError` | 429 | Notice `connectionsFailed` |
| `ConnectionParseError` | 502 | Retry-worthy — **pas** « no connections » |
| Empty `[]` | 200 | Le client distingue épuisé vs vide première fois |

**À COMPRENDRE MAINTENANT :** Un tableau JSON vide et un échec parse 502 ont des **sémantiques différentes**. Vide = « rien à montrer » bien formé. 502 = échec upstream transitoire — ne doit **pas** être caché comme épuisement.

---

## Étape 3 — `getConnections` : orchestration cache-first

**Where :** `lib/ai/graph-service.ts` → `getConnections`

### 3a — Lecture cache

**FACT :** `readActiveRows(fromRef, kind, locale, excludeRefs)` sélectionne depuis `connections` où :

- `from_ref`, `kind`, `locale` correspondent
- `status = 'active'`
- `to_ref NOT IN excludeRefs` (quand fourni)
- ordonné par `toRef`, limit 200

**FACT :** Pour locales non-`en`, un cache hit nécessite que **chaque** ligne anglaise active ait une ligne locale correspondante (porte complétude ref par ref). Des ensembles locale partiels re-entrent le chemin miss.

**FACT :** Sur hit → `hydrate(rows, kind)` → `resolveVerse(toRef)` pour chaque ligne → `ConnectionResult[]`. **Pas d'AI.**

### 3b — Gardes cache miss

| Guard | Purpose |
| --- | --- |
| `consume(`gen:${clientKey}`)` | Rate-limit chemin miss coûteux par IP client |
| `singleFlight(key, factory)` | Misses identiques concurrents partagent une génération |
| `resolveProvider` / `resolveModel` | Provider+model résolus une fois, passés dans key + persist |

**FACT :** `cellKey` = `` `${fromRef}:${kind}:${locale}:${provider}:${model}:${sorted excludeRefs}` ``

**FACT :** Miss non-`en` → `generateLocalizedCell` : assure lignes anglaises existent, puis `translateReason` par ligne manquante (consommation budget séparée). La **sélection** anglaise est toujours canonique ; la locale affecte seulement le **texte raison**.

---

## Étape 4 — `generateConnectionsForCell` : la bifurcation

**Where :** `graph-service.ts` → `generateConnectionsForCell` (exporté pour backfill admin aussi)

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

| `candidates` | `excludeRefs` | Path | AI called? | Typical meaning |
| --- | --- | --- | --- | --- |
| > 0 | any | **Grounded** | Yes | Cas normal |
| 0 | > 0 | **Return []** | No (`calledAI` still true*) | Pool grounded « get more » épuisé |
| 0 | 0 | **Legacy** | Yes | Morphologie/embeddings pas seedés pour verset |

\* **FACT :** `calledAI = candidates.length > 0 || excludeRefs.length === 0` — pool vide sur « get more » marque encore `calledAI: true` pour que backfill admin enregistre l'épuisement correctement.

**À COMPRENDRE MAINTENANT :** Le chemin legacy **ne tourne jamais** quand `excludeRefs.length > 0`. Sans cette règle, « get more » régénérerait des refs similaires depuis mémoire modèle et casserait la pagination.

---

## Étape 5 — `discoverCandidates` : discovery déterministe

**Where :** `lib/ai/connection-discovery.ts`

**FACT :** Limite par défaut = **12** candidats par passe génération.

### Par kind

| Kind | Source table | Algorithm |
| --- | --- | --- |
| `root` | `word_morphology` | Racines distinctes pour ref source → versets partageant ces racines → rank par `count(distinct root)` DESC |
| `thematic` | `verse_embeddings` | `semanticCandidates` → plus proches voisins pgvector |
| `contrast` | `verse_embeddings` | **Même pool voisins que thematic** — AI sélectionne les vraiment opposés |

**FACT :** `semanticCandidates` (`lib/quran/semantic-search.ts`) : charge embedding source → `nearest()` similarité cosinus → retourne refs. Retourne `[]` si source n'a **pas** d'embedding stocké.

**FACT :** `excludeRefs` passé au SQL (`notInArray`) pour chemins root et sémantique.

**FACT :** `[]` vide depuis discovery signifie *« pas de données grounding pour ce verset »* — **pas** une erreur. Les appelants décident fallback vs épuisement.

### Dépendances seeding (offline)

| Data | Script | Needed for |
| --- | --- | --- |
| `verses` | `scripts/seed-quran.mjs` | Toute hydratation |
| `word_morphology` | `scripts/seed-morphology.mjs` | Discovery `root` |
| `verse_embeddings` | `scripts/embed-corpus.mjs` | Discovery `thematic` / `contrast` |

**INFERENCE :** Une DB locale dev fraîche avec Coran seedé mais **sans** embeddings/morphologie touchera **legacy** `generateConnections` au premier expand — jusqu'à ce que tables grounding soient remplies.

---

## Étape 6 — `generateGroundedConnections` : AI articule (chemin préféré)

**Where :** `lib/ai/connection-generator.ts`

### 6a — Hydrater candidats avant prompting

```typescript
const verseMap = await getVerses(candidateRefs);
const candidates = candidateRefs
  .map((ref) => verseMap.get(ref))
  .filter((v) => v !== undefined && v.ref !== fromRef);
if (candidates.length === 0) return [];  // no LLM call
```

**FACT :** Si aucun candidat résout depuis corpus local, retourne `[]` **sans appeler l'AI**.

### 6b — Construction prompt

| Piece | Source |
| --- | --- |
| Base template | `getPrompt("connection.selection", SELECTION_FALLBACK_TEMPLATE)` |
| Task wording | `KIND_SELECTION[kind]` — diffère par kind |
| Candidate list | `- {ref} — {translation}` par verset |
| Maturidi/Hanafi framing | Dans template |
| Rule: choose ONLY from list | Dans template |
| **Tanzih** | `tanzihDirective()` — **ajouté, non-overridable** |
| Locale | `languageDirective(locale)` — ajouté pour non-`en` |

**FACT :** `tanzihDirective()` ajoute `TANZIH_CONSTRAINT` depuis `lib/ai/theological-constraints.ts` :

> *« strict Tanzih (divine transcendence): never describe or imply physical form, spatial location, or resemblance to created things (Tashbih) »*

**FACT :** Les directives Tanzih et langue ne sont **pas** des `{{placeholders}}` dans templates overridables admin — elles sont ajoutées après `renderTemplate` pour que overrides prompt DB ne puissent pas les omettre (voir Phase 3).

### 6c — Appel LLM + log audit

**FACT :** `callAIDetailed(prompt, { feature: "connections", provider, model, ... })`

**FACT :** Insert best-effort dans `ai_generations` (fromRef, kind, model, tokens, promptVersion). Échec log **ne fait pas** échouer la génération.

### 6d — Parse + validate (la porte anti-hallucination)

```typescript
const allowed = new Set<string>(candidates.map((v) => v.ref));
const chosen = parseRawConnections(text)
  .filter((c) => allowed.has(c.ref) && c.ref !== fromRef)
  .slice(0, 3);
```

| Step | Deterministic rule |
| --- | --- |
| `parseRawConnections` | Extrait premier tableau JSON `[...]` ; valide forme ; lance `ConnectionParseError` si malformé |
| `allowed.has(c.ref)` | **Drop toute ref pas dans ensemble discovery** — même si modèle « connaît » un verset valide |
| `c.ref !== fromRef` | Ne connecte jamais verset à lui-même |
| `.slice(0, 3)` | Cap à 3 arêtes par génération |
| Map through `verseMap` | Hydrate `ConnectionResult` complet |

**FACT :** Test prouve modèle retournant `9:99` quand pas dans liste candidats → dropé (`connection-generator.test.ts` : *« rejects any ref the model returns that was not in the candidate set »*).

**FACT :** `[]` bien formé du modèle → sélection vide valide (pas de throw). Prose malformée / refus → `ConnectionParseError` → API 502.

**À COMPRENDRE MAINTENANT :** La validation grounded est **`allowed.has(c.ref)`**, pas seulement `isValidRef`. Une ref fabriquée mais syntaxiquement valide est rejetée si discovery ne l'a pas surfaceée.

---

## Étape 7 — `generateConnections` : repli legacy

**When :** `discoverCandidates` retourne `[]` **et** `excludeRefs.length === 0` (premier miss, pas de données grounding).

**FACT :** Utilise `getPrompt("connection.legacy", LEGACY_FALLBACK_TEMPLATE)` + `KIND_INSTRUCTIONS[kind]`.

**FACT :** Le modèle peut proposer toute ref — filtrée par :

1. `parseRawConnections`
2. `isValidRef(c.ref) && c.ref !== fromRef`
3. `getVerses(candidates.map(c => c.ref))` — **doit exister dans corpus local**
4. `.slice(0, 3)`

**FACT :** Refs qui passent `isValidRef` mais manquent lookup corpus sont dropées silencieusement — pas d'hydratation API externe sur ce chemin.

**INFERENCE :** Legacy est un **chemin bootstrap** pour données grounding non seedées, pas le comportement produit long terme. Production avec seeds complets devrait presque toujours toucher le chemin grounded.

---

## Étape 8 — Persistence et gestion race

**Where :** fin de `generateConnectionsForCell`

**FACT :** Insert dans `connections` :

```typescript
{ fromRef, toRef: g.ref, kind, reason: g.reason, model, locale: "en" }
.onConflictDoNothing()
```

**FACT :** Index unique sur `(from_ref, to_ref, kind, locale)` — générations concurrentes sur deux instances : le perdant re-lit la raison persistée du gagnant.

**FACT :** Échec persist est loggé + `incr("gen_persist_failed")` — l'appelant retourne encore les résultats générés **cette requête**, mais la prochaine requête paie le coût AI à nouveau.

**FACT :** `connections.status` peut être `active | flagged | retired` — file revue admin à `/admin/connections`. Le canvas lit lignes **active** seulement.

---

## Étape 9 — Réponse retour au canvas

**FACT :** `hydrate` / `ConnectionResult[]` mappé inclut : ref, arabicText, translation, surah names, **reason**, **kind**.

**FACT :** Boucle `runExpansion` :

- Tableau vide + `excludeRefs.length > 0` → `ExhaustedExpansionError` → « no more connections »
- Tableau vide + première requête → « no connections found » générique
- Non-vide → placement nœuds décalé (`radialPos`, `findFreeSlot`), `addVerseNode`, `addConnectionEdge`

**FACT :** Si ref cible déjà sur canvas → dessine arête seulement (pas de nœud dupliqué).

---

## Trois kinds arête — même pipeline, discovery + intention sélection différentes

| Kind | Discovery question | AI selection question (`KIND_SELECTION`) |
| --- | --- | --- |
| **thematic** | Qui est sémantiquement proche ? (embeddings) | Choisir 3 qui **partagent thème théologique** |
| **root** | Qui partage racines arabes ? (morphologie) | Choisir 3 où racine partagée **porte sens pertinent** |
| **contrast** | Qui est sémantiquement proche ? (même pool que thematic) | Choisir 3 avec **concept le plus opposé** |

**INFERENCE :** Contrast est le kind le plus subtil — discovery est **basée similarité**, articulation est **basée opposition**. Le modèle doit faire le travail théologique dans la sélection, pas la discovery.

---

## Déterministe vs probabiliste — checklist contributeur

| Step | Deterministic? | Can you "fix a test" by loosening? |
| --- | --- | --- |
| `isValidRef` | Yes | **Never** |
| `discoverCandidates` SQL | Yes | Fix seed data, not bounds |
| `allowed.has(c.ref)` | Yes | **Never** |
| `getVerses` hydration | Yes | Fix corpus, not skip lookup |
| `parseRawConnections` | Yes | Fix prompt/model, not parser |
| `tanzihDirective` append | Yes | **Never** remove |
| Reason wording | No (LLM) | Admin review + prompt changes, with PR disclosure |
| Which 3 of 12 candidates | No (LLM) | Accept variance within validation |

---

## Modes d'échec que vous verrez en développement

| Symptom | Likely cause | Where to look |
| --- | --- | --- |
| Premier expand → vide, pas d'erreur | Pas d'embeddings/morphologie seedés ; legacy n'a rien retourné de valide | Lancer scripts seed ; vérifier retour `discoverCandidates` |
| « Get more » → « no more connections » | Pool grounded épuisé pour kind | Attendu — `excludeRefs` + candidats vides |
| Expand → toast erreur, 502 dans réseau | `ConnectionParseError` — refus/JSON tronqué | Réponse modèle ; retry |
| 429 sur expand | Rate limit sur chemin miss | `lib/infra/rate-limit.ts` ; cache hits non affectés |
| Mêmes connections chaque fois | Cache hit — fonctionne comme prévu | Table `connections` |
| UI turque, raisons anglaises brièvement | Traduction locale en cours / budget épuisé | `generateLocalizedCell` serve partiel |

---

## Tests clés (lisez avant de changer le pipeline)

| File | What it locks |
| --- | --- |
| `__tests__/lib/ai/connection-generator.test.ts` | Erreurs parse, `allowed.has`, hydratation corpus, Tanzih/locale dans prompts |
| `__tests__/lib/ai/connection-discovery.test.ts` | Ranking root, excludeRefs dans SQL, routage kind |
| `__tests__/lib/ai/graph-service.test.ts` | Cache hit/miss, single-flight, excludeRefs, complétude locale |
| `__tests__/integration/graph.integration.test.ts` | End-to-end avec Postgres réel |
| `__tests__/components/canvas/HikmahCanvas.test.tsx` | Comportement client vide vs épuisé |

---

## Fichiers à garder ouverts en debug expand

| File | Role |
| --- | --- |
| `components/canvas/HikmahCanvas.tsx` | `runExpansion`, sémantiques erreur client |
| `store/canvas.ts` | `getExpansionRefs` |
| `app/api/connections/route.ts` | Frontière HTTP + mapping erreur |
| `lib/ai/graph-service.ts` | Cache, single-flight, locale, persist |
| `lib/ai/connection-discovery.ts` | SQL candidats |
| `lib/quran/semantic-search.ts` | Voisins embedding |
| `lib/ai/connection-generator.ts` | Prompts, parse, grounded vs legacy |
| `lib/quran/quran-corpus.ts` | `isValidRef`, `getVerses` |
| `lib/ai/theological-constraints.ts` | `TANZIH_CONSTRAINT` |
| `lib/infra/db/schema.ts` | `connections`, `ai_generations`, indexes |

---

## À COMPRENDRE MAINTENANT

1. **Chemin grounded :** discovery produit refs → AI sélectionne sous-ensemble → `allowed.has(c.ref)` impose le sous-ensemble.
2. **Chemin legacy :** seulement quand discovery vide **et** pas une requête « get more ».
3. **Vide `[]` ≠ échec parse** — statut HTTP différent, UX client différente, sémantiques backfill admin différentes.
4. **La sélection anglaise est canonique** — non-`en` traduit raisons, ne re-choisit pas versets.
5. **Postgres `connections` est le graphe produit** — canvas est layout session ; partage sauve JSON canvas, pas cette table.
6. **Ne jamais assouplir validation verset pour passer un test** — fixez données, seeds ou prompts à la place.

---

## UTILE PLUS TARD

- Boucle backfill admin : `lib/ai/connection-batch.ts`, `connection_coverage.exhaustedAt`, rotation clés Gemini (`AGENTS.md`).
- Overrides prompt admin : `lib/ai/prompt-registry.ts` — peut changer wording, pas append Tanzih.
- Sélection provider : `lib/ai/ai.ts` → `resolveProvider("connections")`.
- Setup test intégration avec Testcontainers pour vérification chemin miss complet.

---

## IGNORER POUR L'INSTANT

- Dimensions exactes modèle embedding et classification quota Gemini (`lib/ai/gemini-errors.ts`) — territoire Phase 9.
- Workflow UI revue connection admin — sauf si vous contribuez outillage modération.
- Comptes dénormalisés `connection_coverage` — bookkeeping backfill seulement.

---

## Questions checkpoint Phase 7

Répondez de mémoire, puis vérifiez dans le code :

1. Quelle fonction impose que le modèle ne peut pas introduire une ref hors de l'ensemble discovery ?
2. Sous quelles deux conditions `generateConnections` (legacy) tourne ?
3. Pourquoi contrast utilise `semanticCandidates` au lieu d'une requête SQL séparée ?
4. Que se passe-t-il si le modèle retourne `[]` bien formé vs refus en prose ?
5. Où Tanzih est injecté, et pourquoi ce n'est pas un `{{placeholder}}` ?

---

**Next :** [Phase 8 — State and canvas model (detailed)](./phase-8-state-and-canvas.md) — stores Zustand (état client), persistence, sérialisation share, et comment l'état canvas se rapporte à (mais n'est pas) le graphe Postgres.

Dites **« continue to Phase 8 »** quand vous êtes prêt, ou demandez de zoomer sur une étape ci-dessus.

> **Version complète (français B2+) :** [Phase 7](../onboarding-fr/phase-7-grounded-ai-connections.md)

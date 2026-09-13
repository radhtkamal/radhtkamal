# Phase 6 — Parcours runtime core

> **Prérequis :** Phases [1](./phase-1-what-is-openhikmah.md)–[5](./phase-5-repository-tour.md)  
> **Tags d'évidence :** **FACT** · **INFERENCE** · **UNKNOWN**

Cinq flux qui enseignent la plupart de l'architecture. Chaque étape nomme des **symboles et chemins exacts**. Lisez un flux à la fois. Tracez-le dans le dépôt avant de continuer.

**Légende pour chaque étape :**

| Champ | Signification |
| --- | --- |
| **Where** (où) | Fichier / fonction |
| **Input** (entrée) | Ce qui arrive |
| **Decision** (décision) | Branche prise |
| **Output** (sortie) | Ce qui change |
| **Next** (suivant) | Qui s'exécute ensuite |

---

## Flux A — Recherche par référence ou mot-clé

**Histoire utilisateur :** Tapez `2:255` ou `mercy` dans le dialog de recherche, choisissez un résultat, arrivez sur le canvas avec un nouveau nœud verset.

### A1 — Le client attend et choisit le mode fetch

| | |
| --- | --- |
| **Where** | `components/search/SearchDialog.tsx` — `useEffect` sur `query` (lignes ~121–140) |
| **Input** | Chaîne `query` depuis l'input |
| **Decision** | Si `/^\d+:\d+$/` → `fetchPreview(q)` après 250ms ; sinon → `fetchSearch(q)` après 420ms |
| **Output** | AbortController annule les requêtes obsolètes |
| **Next** | `fetchPreview` ou `fetchSearch` |

**FACT :** Input en forme de ref montre preview inline ; tout le reste lance recherche complète.

### A2a — Chemin preview référence (client)

| | |
| --- | --- |
| **Where** | `SearchDialog.fetchPreview` |
| **Input** | `"2:255"` |
| **Decision** | `GET /api/verse/2/255` |
| **Output** | `previewVerse: Verse` dans l'état composant |
| **Next** | L'utilisateur appuie Entrée ou clique pour sélectionner (même fetch verset dans `selectResult`) |

### A2b — Chemin recherche mot-clé (client → API)

| | |
| --- | --- |
| **Where** | `SearchDialog.fetchSearch` → `GET /api/search?q=...` |
| **Input** | Requête texte libre |
| **Output** | `SearchResponse` → `searchResults`, `relatedResults`, `totalResults` |
| **Next** | `app/api/search/route.ts` `GET` |

### A3 — Routage API recherche (serveur)

| | |
| --- | --- |
| **Where** | `app/api/search/route.ts` `GET` |
| **Input** | `q`, cookies → `getQuranEdition()`, `getUiLocale()` |
| **Decision tree** | |

```text
1. Exact surah name match?     → return { matchedSurahs } (no ayah list)
2. q matches /^\d+:\d+$/ ?
   → isValidRef(q)?             → resolveVerse(q) or empty results
3. Else keyword path:
   → consume(`searchkw:...`)   → rate limit
   → Promise.all([
        keywordSearch(...),     → quran.com API → hydrate via getVerses
        relatedByMeaning(...)   → searchByMeaning (page 1 only)
     ])
```

| **Output** | JSON `SearchResponse` avec `results`, optionnel `related`, optionnel `matchedSurahs` |
| **Next** | Le client affiche les listes |

**FACT :** Les hits mot-clé viennent de **quran.com** ; l'arabe/traduction affiché vient du **corpus local** (`hydrate()` → `getVerses`).

**FACT :** Ref invalide comme `1:8` ou `02:255` → **résultats vides**, jamais une carte inventée.

### A4 — L'utilisateur sélectionne un résultat

| | |
| --- | --- |
| **Where** | `SearchDialog.selectResult` ou `loadSeedVerse` |
| **Input** | `SearchResult.ref` |
| **Decision** | `GET /api/verse/${surah}/${ayah}` |
| **Output** | Objet `Verse` complet |
| **Next** | `mapConnections(verse)` |

### A5 — API verset (porte serveur)

| | |
| --- | --- |
| **Where** | `app/api/verse/[surah]/[ayah]/route.ts` `GET` |
| **Input** | `ref = "${surah}:${ayah}"` |
| **Decision** | `isValidRef(ref)` → 400 si false |
| **Algorithm** | `resolveVerse(ref, edition)` — corpus d'abord, fallback alquran.cloud |
| **Output** | JSON `Verse` ou 404 |
| **Next** | Client `mapConnections` |

### A6 — Placer nœud sur canvas (état client)

| | |
| --- | --- |
| **Where** | `SearchDialog.mapConnections` |
| **Input** | `Verse`, `nodes` actuels, `viewport` |
| **Algorithm** | Premier nœud → `{x:0,y:0}` + `setPendingAutoExpand(nodeId)` ; sinon `findFreeSlot` + `setPendingPanToNode(nodeId)` |
| **Output** | `useCanvasStore.addVerseNode({ ...verse, isRoot: isFirst }, position)` → nouvel id `node-N` |
| **Next** | Effets `HikmahCanvas` (auto-expand Flux C) + autosave `useCanvasPersistence` |

```mermaid
sequenceDiagram
  participant U as User
  participant SD as SearchDialog
  participant API as /api/search
  participant V as /api/verse/s/a
  participant Z as useCanvasStore

  U->>SD: type query
  SD->>API: GET q=mercy
  API-->>SD: results + related
  U->>SD: select result
  SD->>V: GET verse
  V-->>SD: Verse JSON
  SD->>Z: addVerseNode
  Note over Z: pendingAutoExpand if first node
```

**À COMPRENDRE MAINTENANT :** Search **trouve** les refs ; l'API verset **valide et hydrate** ; le store canvas **possède le layout**, pas la DB graphe.

---

## Flux B — Recherche par sens (couche sémantique)

**Histoire utilisateur :** Tapez un concept en anglais ; voyez des hits « Related by meaning » même si les mots-clés diffèrent.

Ce n'est **pas** un mode recherche séparé dans l'UI — il s'exécute **en parallèle** de la recherche mot-clé sur la page 1.

### B1 — Déclencheur (serveur, dans la route search)

| | |
| --- | --- |
| **Where** | `app/api/search/route.ts` → `relatedByMeaning(req, q, edition)` |
| **Input** | Même `q` que recherche mot-clé, `page === 1` seulement |
| **Decision** | `consume(`search:${clientKey}`)` — partage le préfixe budget génération AI |
| **Decision** | Timeout 4s via `AbortController` |
| **Next** | `searchByMeaning(q, limit, edition, signal)` |

**FACT :** Échec → `[]` silencieusement ; l'UI omet la section « Related by meaning ».

### B2 — Embed query + plus proches voisins

| | |
| --- | --- |
| **Where** | `lib/quran/semantic-search.ts` |
| **Input** | Chaîne query normalisée |
| **Algorithm** | |

```text
embedQueryCached(q)
  → Redis hit? return vector
  → else embed(q) via Gemini (lib/ai/ai.ts)
  → cache 7 days

nearest(queryVec, limit)
  → SQL: 1 - cosineDistance(verse_embeddings.embedding, queryVec)
  → ORDER BY similarity DESC

hydrate(rows, edition)
  → getVerses(refs, edition)
```

| **Output** | `SemanticMatch[]` avec `verse` + `similarity` |
| **Next** | Route search déduplique contre refs mot-clé → `response.related` |

### B3 — Affichage client

| | |
| --- | --- |
| **Where** | `SearchDialog` — `setRelatedResults(data.related ?? [])` |
| **Input** | `SearchResult[]` dans `related` |
| **Output** | Section UI séparée « Related by meaning » |
| **Next** | Même `selectResult` → `/api/verse/...` → canvas comme Flux A |

### Lié : « Versets similaires » depuis sidebar (entrée différente)

| | |
| --- | --- |
| **Where** | `GET /api/verse/[surah]/[ayah]/similar` → `similarVerses(ref, 5)` |
| **Input** | Ref verset source (doit passer `isValidRef`) |
| **Algorithm** | Charge embedding source depuis `verse_embeddings` ; plus proches voisins excluant soi |
| **Output** | `{ verse, similarity }[]` — vide si pas d'embedding seedé |

**INFERENCE :** Le classement sémantique est **indexé anglais** (embeddings construits depuis texte traduction) ; l'affichage utilise le cookie **edition** de l'utilisateur.

---

## Flux C — Expansion d'un verset sur le canvas

**Histoire utilisateur :** Cliquez **+** sur un nœud → choisissez Theme / Root / Contrast → nouveaux nœuds et arêtes apparaissent avec raisons AI.

C'est la **boucle produit centrale** — lisez lentement.

### C1 — L'utilisateur sélectionne le type d'expansion

| | |
| --- | --- |
| **Where** | `components/canvas/VerseNode.tsx` → `handleExpandSelect(kind)` |
| **Input** | `EdgeKind` depuis `ExpandMenu` |
| **Output** | `setPendingExpand({ nodeId, ref: verse.ref, kind })` |
| **Next** | `HikmahCanvas` `useEffect` sur `pendingExpand` |

**FACT :** Le premier nœud search auto-expand **thematic** via `setPendingAutoExpand` → même chemin `runExpansion` avec `kind: "thematic"`.

### C2 — L'effet expansion se déclenche

| | |
| --- | --- |
| **Where** | `components/canvas/HikmahCanvas.tsx` lignes ~253–261 |
| **Input** | `pendingExpand` |
| **Algorithm** | Efface pending ; charge nœud source ; lit `verse.arabicText`, `verse.translation`, position |
| **Output** | Appelle `runExpansion(nodeId, ref, kind, arabicText, translation, sourcePos)` |
| **Next** | POST réseau |

### C3 — Le client collecte la liste exclude (« get more »)

| | |
| --- | --- |
| **Where** | `runExpansion` → `getExpansionRefs(nodeId, kind)` |
| **Input** | Arêtes canvas actuelles |
| **Algorithm** | Filtre arêtes où `e.source === nodeId && e.data.kind === kind` → map refs versets cibles |
| **Output** | `excludeRefs: string[]` envoyé à l'API |
| **Next** | `POST /api/connections` |

### C4 — Validation frontière API

| | |
| --- | --- |
| **Where** | `app/api/connections/route.ts` `POST` |
| **Input** | Corps JSON |
| **Decisions** | Champs requis présents ; `kind` ∈ ensemble autorisé ; `isValidRef(fromRef)` ; chaque entrée `excludeRefs` valide ; longueur texte ≤ 5000 |
| **Output** | Appelle `getConnections(fromRef, kind, { arabicText, translation }, { clientKey, excludeRefs, locale })` |
| **Next** | `lib/ai/graph-service.ts` |

### C5 — Graph service : chemin cache hit

| | |
| --- | --- |
| **Where** | `getConnections` → `readActiveRows(fromRef, kind, locale, excludeRefs)` |
| **Input** | Clé cellule + locale |
| **Algorithm** | `SELECT * FROM connections WHERE from_ref, kind, locale, status='active', to_ref NOT IN excludeRefs` |
| **Decision** | Si lignes existent et porte complétude locale passe → `hydrate(rows, kind)` |
| **Algorithm (hydrate)** | Pour chaque ligne : `resolveVerse(toRef)` → construit `ConnectionResult` |
| **Output** | `ConnectionResult[]` — **pas d'appel AI** |
| **Next** | Réponse JSON au client |

### C6 — Graph service : chemin cache miss (résumé)

| | |
| --- | --- |
| **Where** | Branche miss `getConnections` |
| **Input** | Même cellule |
| **Decisions** | `consume(`gen:${clientKey}`)` rate limit ; `singleFlight(key, ...)` déduplique requêtes identiques concurrentes |
| **Output** | `generateConnectionsForCell(...)` ou `generateLocalizedCell(...)` pour non-`en` |
| **Next** | Phase 7 tracera AI en détail ; pour Phase 6 connaissez la chaîne : |

```text
discoverCandidates(fromRef, kind, undefined, excludeRefs)
  root     → rootCandidates()        SQL on word_morphology
  thematic → semanticCandidates()    pgvector neighbors
  contrast → semanticCandidates()    same pool; AI picks opposites

if candidates.length > 0
  generateGroundedConnections(..., candidateRefs)
else if excludeRefs.length > 0
  return []   // "get more" exhausted
else
  generateConnections(...)   // legacy, corpus-validated

INSERT connections ... ON CONFLICT DO NOTHING
return hydrate / map to ConnectionResult[]
```

### C7 — Le client applique les résultats au canvas

| | |
| --- | --- |
| **Where** | Boucle `HikmahCanvas.runExpansion` |
| **Input** | `ConnectionResult[]` |
| **Decision** | `connections.length === 0` → notice erreur (épuisé vs échec première fois) |
| **Per connection** | |

```text
if hasNode(conn.ref)
  → buildConnectionEdge(existingNode) → addConnectionEdge
else
  → radialPos + findFreeSlot → addVerseNode(conn)
  → buildConnectionEdge → addConnectionEdge
```

| **Output** | `nodes`, `edges` mis à jour dans Zustand ; animation décalée 350ms ; `reactFlow.fitView` |
| **Next** | Sauvegarde localStorage débouncée `useCanvasPersistence` (~800ms) |

### C8 — Clic arête → sidebar

| | |
| --- | --- |
| **Where** | `HikmahCanvas.handleEdgeClick` |
| **Input** | React Flow `edge` |
| **Output** | `setSidebarContent({ type: "edge", fromVerse, toVerse, reason, kind, label })` |
| **Next** | `ContextSidebar` affiche raison AI (style éditorial, pas scripture) |

```mermaid
flowchart TD
  VM[VerseNode ExpandMenu] --> PE[setPendingExpand]
  PE --> RE[runExpansion]
  RE --> API[POST /api/connections]
  API --> GC[getConnections]
  GC --> HIT{cache hit?}
  HIT -->|yes| HY[hydrate rows]
  HIT -->|no| DISC[discoverCandidates]
  DISC --> AI[generateGroundedConnections]
  AI --> INS[INSERT connections]
  INS --> HY
  HY --> JSON[ConnectionResult array]
  JSON --> RE
  RE --> ZS[addVerseNode + addConnectionEdge]
```

**À COMPRENDRE MAINTENANT :** `excludeRefs` est comment « expand again » évite de répéter les mêmes cibles. Tableau vide au premier miss ≠ erreur parse (502 depuis `ConnectionParseError`).

---

## Flux D — Partager un canvas

**Histoire utilisateur :** Cliquez Share dans la toolbar → URL copiée → le destinataire ouvre le lien → voit le même layout.

### D1 — Sérialiser et POST

| | |
| --- | --- |
| **Where** | `components/canvas/CanvasToolbar.tsx` `handleShare` |
| **Input** | `nodes`, `edges` depuis Zustand |
| **Algorithm** | `serializeCanvas(nodes, edges)` → `SavedCanvas { v:1, nodes, edges }` |
| **Output** | `buildShareUrl(saved)` |
| **Next** | `hooks/useCanvasPersistence.ts` |

### D2 — Persister snapshot partage

| | |
| --- | --- |
| **Where** | `buildShareUrl` → `POST /api/share` |
| **Input** | Corps JSON |
| **Decisions** | Taille ≤ 512KB ; `v === 1` ; chaque nœud passe `isValidNode()` (ref, surahName, chaînes translation) |
| **Algorithm** | `crypto.randomUUID()` → `INSERT shared_canvases { id, data }` |
| **Output** | `{ id }` → URL `/canvas?share=<uuid>` |
| **Next** | Clipboard via `useCopyFeedback.copy` |

**FACT :** Share stocke **layout + snapshots versets**, pas le graphe Postgres `connections`.

### D3 — Le destinataire restaure

| | |
| --- | --- |
| **Where** | Effet mount `useCanvasPersistence` |
| **Input** | `?share=<uuid>` dans l'URL |
| **Algorithm** | Valide regex UUID → `GET /api/share/[id]` → parse JSON |
| **Output** | `restoreCanvas(saved)` dans `store/canvas.ts` |
| **Algorithm (restore)** | `deserializeCanvas` → reset compteur id nœud → bump `restoreToken` |
| **Side effect** | Retire param `share` de l'URL au succès ; fallback localStorage si fetch échoue |
| **Next** | React Flow affiche graphe restauré |

### D4 — GET share API

| | |
| --- | --- |
| **Where** | `app/api/share/[id]/route.ts` `GET` |
| **Input** | UUID |
| **Decisions** | Rate limit ; format UUID ; ligne existe |
| **Output** | JSON `SavedCanvas` parsé |

**IGNORER POUR L'INSTANT :** Route image OG `app/api/share/[id]/opengraph-image.tsx` — preview social seulement.

---

## Flux E — Marquer un verset en favori (invité + connecté)

**Histoire utilisateur :** Basculez favori sur un verset — fonctionne offline en invité ; sync vers Postgres quand connecté.

### E1 — Bascule client optimiste

| | |
| --- | --- |
| **Where** | `store/auth.ts` → `toggleBookmark(ref)` |
| **Input** | Chaîne ref verset |
| **Algorithm** | Ajout/suppression optimiste dans `bookmarks[]` ; set `bookmarkBusy[ref]` |
| **Decision** | Pas de `accessToken` → stop (persist local seulement via Zustand `persist`) |
| **Decision** | Token présent → `POST /api/bookmarks` ou `DELETE /api/bookmarks/[ref]` |
| **Output** | Au succès : update `pendingBookmarkAdds` ; à l'échec : rollback |
| **Next** | Frontière auth API |

### E2 — API favoris

| | |
| --- | --- |
| **Where** | `app/api/bookmarks/route.ts` |
| **Input** | Bearer token + `{ ref }` |
| **Decisions** | `requireUser(req)` → 401 si absent ; `isValidRef(ref)` → 400 |
| **Algorithm** | `INSERT bookmarks ON CONFLICT DO NOTHING` |
| **Output** | `{ ok: true }` |
| **Next** | Le client efface l'état busy |

### E3 — Sync restauration session

| | |
| --- | --- |
| **Where** | `components/providers.tsx` `SessionRestorer` → `loadRemoteBookmarks()` |
| **Input** | Access token depuis `/api/auth/refresh` |
| **Algorithm** | `GET /api/bookmarks` → merge refs serveur avec local ; sync `pendingBookmarkAdds` via POSTs concurrents bornés |
| **Output** | `bookmarks[]` unifié ; refs invité uploadées une fois |
| **Next** | Page favoris lit le même store |

**FACT :** Le store favoris garde **refs seulement** — pas le texte verset complet (texte fetché via `/api/verse/...` à l'affichage).

**Contraste avec workspace (variante Flux D) :** `CanvasToolbar.handleSave` → `POST /api/workspace` avec `serializeCanvas` complet — nécessite auth ; persiste canvas nommé dans `saved_workspaces`.

---

## Comparaison inter-flux

| Flux | Touche graphe Postgres ? | Touche AI ? | Store client principal |
| --- | --- | --- | --- |
| A Search | Non | Seulement via B `related` | `canvas` (à la sélection) |
| B Semantic | Lit `verse_embeddings` | Embed query (Gemini) | — |
| C Expand | **Oui** `connections` | Sur cache miss | `canvas` |
| D Share | `shared_canvases` seulement | Non | `canvas` + URL |
| E Bookmark | `bookmarks` | Non | `auth` |

---

## Comportement erreur à connaître

| Symptôme | Cause probable | Où affiché |
| --- | --- | --- |
| Search 429 | Rate limit `searchkw:` | SearchDialog `searchError: "rateLimited"` |
| Search vide pour ref valide en apparence | `isValidRef` ou `resolveVerse` null | Résultats vides, pas erreur |
| Expand « connections failed » | API non-OK ou Error lancée | Notice `HikmahCanvas` |
| Expand « no more connections » | `excludeRefs` non-vide + tableau vide | `ExhaustedExpansionError` |
| Expand 502 | `ConnectionParseError` (mauvais JSON AI) | Notice échec générique ; retry |
| Share échoue | POST rejeté / réseau | Flash erreur share toolbar |
| Favori rollback | POST/DELETE échoué | Ref retirée de la liste locale |

---

## Phase 6 — Ce qu'il faut retenir

### À COMPRENDRE MAINTENANT

1. **Search** = trouver refs (mot-clé/ref/nom sourate) ; **API verset** = valider + hydrater ; **canvas** = layout.
2. **Sémantique** = embed query + pgvector ; complémentaire au mot-clé sur page recherche 1.
3. **Expand** = `pendingExpand` → `runExpansion` → `getConnections` → cache ou discover+AI → `addVerseNode`/`addConnectionEdge`.
4. **Share** = sérialiser layout vers `shared_canvases`, pas le graphe global.
5. **Favoris** = liste refs dans store auth ; invité local, connecté sync.

### UTILE PLUS TARD

- Sémantique merge `appendWorkspace` au chargement workspace sauvegardé
- `restoreToken` pour déduplication activity tracker sur restore bulk
- Backfill admin écrit les mêmes lignes `connections` sans UI

### IGNORER POUR L'INSTANT

- Audio `playGraph` depuis toolbar
- Pipeline export PNG/PDF

---

## Incertitudes

| Sujet | Statut |
| --- | --- |
| Nombres exacts rate-limit | **FACT :** dans `lib/infra/rate-limit.ts` — lire quand debug 429 |
| Si snapshot share inclut arabe complet sur tous nœuds | **FACT :** `SavedNode.verse` est objet `Verse` complet au moment sérialisation |

---

## Et ensuite

**Phase 7 — Connections AI grounded (plongée profonde) :** pipeline complet pour empêcher refs inventées — le modèle de confiance central en détail pas à pas avec son propre diagramme runtime.

Dites **« continue to Phase 7 »** quand vous êtes prêt.

---

## Checklist trace (faites-le vous-même)

Ouvrez ces fichiers côte à côte en relisant un flux :

| Flux | Fichiers à garder ouverts |
| --- | --- |
| A | `SearchDialog.tsx`, `app/api/search/route.ts`, `app/api/verse/.../route.ts`, `store/canvas.ts` |
| B | `app/api/search/route.ts`, `lib/quran/semantic-search.ts`, `lib/ai/ai.ts` |
| C | `VerseNode.tsx`, `HikmahCanvas.tsx`, `app/api/connections/route.ts`, `lib/ai/graph-service.ts` |
| D | `CanvasToolbar.tsx`, `useCanvasPersistence.ts`, `app/api/share/route.ts`, `store/canvas.ts` |
| E | `store/auth.ts`, `app/api/bookmarks/route.ts`, `components/providers.tsx` |

> **Version complète (français B2+) :** [Phase 6](../onboarding-fr/phase-6-runtime-walkthroughs.md)

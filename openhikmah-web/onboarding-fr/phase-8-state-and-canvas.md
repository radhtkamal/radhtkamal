# Phase 8 — Modèle d'état et canvas

> **Prérequis :** Phases [1](./phase-1-what-is-openhikmah.md)–[7](./phase-7-grounded-ai-connections.md)  
> **Tags d'évidence :** **FACT** (fait vérifiable dans le code) · **INFERENCE** (déduction raisonnée) · **UNKNOWN** (non documenté ou incertain)

La Phase 7 a tracé comment les connexions IA sont **générées et mises en cache dans Postgres**. La Phase 8 répond : **ce que l'utilisateur voit et manipule réellement sur le canvas infini** — et comment cet état de session est stocké, partagé et restauré.

**Phrase centrale (à mémoriser) :**

> Le canvas est un **graphe de layout de session** ; Postgres `connections` est le **graphe de connaissance produit**. Ils se chevauchent en signification mais pas en stockage.

---

## Trois graphes (ne pas les confondre)

| Graphe | Où | Ce qu'il stocke | Survit au refresh ? |
| --- | --- | --- | --- |
| **Postgres `connections`** | DB serveur | Arêtes générées IA : `(fromRef, toRef, kind, reason, locale)` | Oui — partagé par tous les utilisateurs |
| **Zustand canvas** | Mémoire navigateur | Nœuds/arêtes React Flow + état chrome UI | Jusqu'à fermeture d'onglet sauf autosave |
| **JSON `SavedCanvas`** | localStorage / `shared_canvases` / `saved_workspaces` | Layout sérialisé + snapshots versets + raisons arêtes | Selon le canal |

**FACT :** Expand un verset appelle `POST /api/connections` (Phase 7), puis **copie** le résultat dans Zustand. Le canvas ne lit pas Postgres `connections` directement au rendu.

**INFERENCE :** Deux utilisateurs avec le même verset source peuvent voir les mêmes raisons IA (cache hit) mais des **layouts différents** sauf s'ils partagent un snapshot canvas.

---

## Architecture : qui possède quoi

```mermaid
flowchart TB
  subgraph zustand ["Zustand — useCanvasStore (store/canvas.ts)"]
    N[nodes: Node[]]
    E[edges: Edge[]]
    UI[pendingExpand, sidebarContent, viewport, ...]
    DERIV[duplicateNodeIdsByRef, expansionCountsByNode]
  end

  subgraph xyflow ["@xyflow/react — HikmahCanvas.tsx"]
    RF[Composant contrôlé ReactFlow]
    ONC[onNodesChange / onEdgesChange]
    RENDER[Rendu VerseNode + HikmahEdge]
  end

  subgraph persist ["Canaux de persistance"]
    LS[(localStorage open-hikmah-canvas)]
    SHARE[(shared_canvases Postgres)]
    WS[(saved_workspaces Postgres)]
  end

  subgraph layout ["Layout — lib/canvas/canvas-layout.ts"]
    FFS[findFreeSlot]
    BCE[buildConnectionEdge]
    VC[viewportCenter]
  end

  N --> RF
  E --> RF
  RF --> ONC --> N
  ONC --> E
  serializeCanvas --> LS
  serializeCanvas --> SHARE
  serializeCanvas --> WS
  LS --> restoreCanvas
  SHARE --> restoreCanvas
  WS --> appendWorkspace
  FFS --> addVerseNode
  BCE --> addConnectionEdge
```

| Couche | Possède | Ne possède PAS |
| --- | --- | --- |
| **Zustand** | Source de vérité pour nodes, edges, sélection, file expansion, payload sidebar | Appels LLM, graphe Postgres, physique drag interne React Flow |
| **React Flow** | Rendu pan/zoom, gestes drag nœud, chemins arêtes, minimap | Texte verset, génération IA, format persistance |
| **`canvas-layout.ts`** | Placement sans collision, construction id arête | Mutations store (les appelants invoquent le store après calcul position) |
| **`useCanvasPersistence`** | Autosave localStorage, restore `?share=`, merge invité→workspace | Persistance viewport (non sauvegardé) |

---

## Qu'est-ce qu'un nœud ?

### Objet domaine

**FACT :** Un nœud canvas représente une **instance placée** d'un `Verse` (`types/quran.ts`) :

```typescript
interface Verse {
  surah: number;
  ayah: number;
  ref: VerseRef;           // "2:255"
  arabicText: string;
  translation: string;
  surahName: string;
  surahNameArabic: string;
  isRoot?: boolean;        // style premier nœud/recherche
  isLoading?: boolean;
}
```

### Enveloppe React Flow

**FACT :** Stocké comme `Node` `@xyflow/react` :

| Champ | Valeur |
| --- | --- |
| `id` | `node-${n}` — compteur client monotone (`nextId()` dans `store/canvas.ts`) |
| `type` | `"verse"` → rend `VerseNode` |
| `position` | `{ x, y }` coordonnées flow |
| `data` | Champs `Verse` étalés |

### ID nœud vs identité verset

| Concept | Clé | Notes |
| --- | --- | --- |
| **ID nœud** | `node-42` | Éphémère par session canvas ; remappé sur collision `appendWorkspace` |
| **Identité verset** | `verse.ref` (`"2:255"`) | Référence Coran canonique ; **la même ref peut apparaître sur plusieurs nœuds** |

**FACT :** `duplicateNodeIdsByRef` mappe chaque `ref` → tous les ids nœuds la partageant. `VerseNode` affiche un indicateur de doublon quand un autre nœud a la même ref.

**FACT :** `hasNode(ref)` retourne true si **n'importe quel** nœud porte cette ref — utilisé pour éviter d'empiler des doublons depuis URL/deep links, mais l'expansion peut **dessiner une arête** vers un nœud existant au lieu d'en créer un second.

---

## Qu'est-ce qu'une arête ?

### Objet domaine

**FACT :** `CanvasEdge` (`types/quran.ts`) :

```typescript
interface CanvasEdge {
  id: string;
  source: string;   // id nœud source
  target: string;   // id nœud cible
  type: "hikmah";
  data: { kind: EdgeKind; label: string; reason?: string };
}
```

| Champ | Signification |
| --- | --- |
| `kind` | `"thematic" \| "root" \| "contrast"` — pilote couleur + comptage expansion |
| `label` | Raison tronquée (≤60 car.) affichée sur la pastille arête |
| `reason` | Explication IA complète — affichée dans `ContextSidebar` au clic arête |

### ID arête et déduplication

**FACT :** `buildConnectionEdge(sourceNodeId, targetNodeId, conn)` (`lib/canvas/canvas-layout.ts`) :

```typescript
id: `edge-${sourceNodeId}-${targetNodeId}`
```

**FACT :** Les arêtes sont **dirigées** en stockage (`source` → `target`) mais `addConnectionEdge` traite `(A,B)` et `(B,A)` comme doublons :

| Résultat | Signification |
| --- | --- |
| `"added"` | Nouvelle arête insérée |
| `"duplicate-same-kind"` | Même paire + même kind déjà existante — skip silencieux |
| `"duplicate-different-kind"` | Même paire, kind différent — notice UI dans `runExpansion` |

**FACT :** `getExpansionRefs(nodeId, kind)` filtre `e.source === nodeId && e.data.kind === kind` — seules les arêtes d'expansion **sortantes** comptent pour la liste exclude « get more » (Phase 7).

---

## Carte des champs store Zustand

**Where:** `store/canvas.ts` → `useCanvasStore`

### Données graphe

| Champ | Objectif |
| --- | --- |
| `nodes`, `edges` | État contrôlé React Flow |
| `onNodesChange`, `onEdgesChange` | Appliquer drag/select/delete via `applyNodeChanges` / `applyEdgeChanges` |

### Index dérivés (recalculés à la mutation)

| Champ | Objectif |
| --- | --- |
| `duplicateNodeIdsByRef` | Détection doublon O(1) par ref |
| `expansionCountsByNode` | Comptes arêtes sortantes par kind (badges menu expand) |

### Orchestration UI / async (non sérialisé)

| Champ | Objectif |
| --- | --- |
| `selectedNodeId` | Sélection courante |
| `expandingNodeId` | Affiche spinner sur nœud pendant `runExpansion` |
| `openExpandNodeId` | Quel menu expand de nœud est ouvert |
| `sidebarContent` | Payload `ContextSidebar` (détail nœud ou arête) |
| `pendingExpand` | `{ nodeId, ref, kind }` — consommé par effet `HikmahCanvas` |
| `pendingAutoExpand` | Expand thématique premier nœud après recherche |
| `pendingPanToNodeId` | Pan caméra après ajout recherche |
| `newlyAddedNodeId` | Animation pulse sur nœud frais |
| `viewport` | `{ x, y, zoom }` — mis à jour au pan/zoom |
| `sidebarWidth` | Sidebar redimensionnable |
| `fitRequestToken` | Signal inter-arbre pour `fitView()` (barre mobile) |
| `restoreToken` | Incrémenté sur restore bulk — activity tracker distingue restore vs ajout utilisateur |

### Mutations que vous toucherez comme contributeur

| Action | Fonction | Appelant typique |
| --- | --- | --- |
| Ajouter un verset | `addVerseNode(verse, position?)` | Recherche, expansion, URL `?verse=` |
| Ajouter colonne sourate complète | `addSurahNodes(verses[])` | URL `?surah=` |
| Ajouter arête IA | `addConnectionEdge(edge)` | `runExpansion` |
| Effacer canvas | `reset()` | Clear toolbar (confirmation armée) |
| Remplacer graphe | `restoreCanvas(saved)` | Lien share, localStorage |
| Fusionner graphe | `appendWorkspace(saved)` | Charger workspace sauvegardé |

---

## Responsabilités React Flow

**Where:** `components/canvas/HikmahCanvas.tsx`

**FACT :** `HikmahCanvas` enveloppe `ReactFlowProvider` → `CanvasInner`.

| Enregistrement | Composant |
| --- | --- |
| `nodeTypes.verse` | `VerseNode` |
| `edgeTypes.hikmah` | `HikmahEdge` |

**FACT :** React Flow est **contrôlé** : `nodes={nodes}` et `edges={edges}` depuis Zustand ; les changements reviennent via `onNodesChange` / `onEdgesChange`.

**FACT :** `HikmahCanvas` est importé dynamiquement avec `ssr: false` dans `CanvasPageClient.tsx` — le canvas nécessite APIs navigateur et React Flow.

**FACT :** `onlyRenderVisibleElements` activé pour performance sur grands graphes.

### Ce que React Flow fait vs Zustand

| Geste utilisateur | React Flow | Zustand |
| --- | --- | --- |
| Drag nœud | Émet changement position | `onNodesChange` met à jour `nodes` |
| Pan/zoom | Met à jour transform interne | `onMove` → `setViewport` |
| Clic arête | `onEdgeClick` | `setSidebarContent({ type: "edge", ... })` |
| Clic pane | `onPaneClick` | Ferme menu expand |

**INFERENCE :** React Flow possède la **physique d'interaction éphémère** ; Zustand possède l'**état graphe durable** que la logique app lit.

---

## Responsabilité layout

**Where:** `lib/canvas/canvas-layout.ts`

| Helper | Utilisé quand |
| --- | --- |
| `findFreeSlot(existing, anchor)` | Recherche spirale position sans chevauchement (nœud 288×240 + gap 48px) |
| `viewportCenter(viewport, w, h)` | Ajout recherche ancré au centre visible |
| `radialPos(source, i, total)` | Expansion étale enfants en arc (`HikmahCanvas.tsx`) |
| `buildConnectionEdge(...)` | Construit id arête + troncature label |

**FACT :** Le placement expansion lit la position source **live** à chaque itération — dragger pendant expansion déplace l'origine du fan.

**FACT :** Les milieux de label arête sont traités comme obstacles layout pour que les nouveaux nœuds ne couvrent pas les pastilles raison IA.

---

## Sérialisation : `SavedCanvas`

**Where:** `store/canvas.ts` → `serializeCanvas` / `deserializeCanvas`

```typescript
interface SavedCanvas {
  v: 1;
  nodes: SavedNode[];  // { id, x, y, verse }
  edges: SavedEdge[];  // { id, source, target, kind, label, reason }
}
```

### Ce qui est inclus

| Inclus | Exclu |
| --- | --- |
| Ids nœuds, positions, snapshots `Verse` complets | `viewport` (pan/zoom) |
| Extrémités arêtes, kind, label, reason | État UI (sidebar, sélection, files pending) |
| Version `v: 1` | Ids lignes connexion Postgres |

**FACT :** Le texte verset dans le JSON sauvegardé est un **snapshot au moment de la sauvegarde** — si les traductions corpus sont mises à jour côté serveur, le canvas restauré affiche encore l'ancien texte jusqu'à re-fetch des nœuds (pas de re-hydratation automatique au restore).

**UNKNOWN :** Si les mainteneurs prévoient une re-sync corpus au restore — non implémenté aujourd'hui.

---

## Canaux de persistance

### 1 — localStorage (invité + connecté)

**Where:** `hooks/useCanvasPersistence.ts`

| Détail | Valeur |
| --- | --- |
| Clé | `open-hikmah-canvas` (`CANVAS_STORAGE_KEY`) |
| Déclencheur | Debounced 800 ms après changement `nodes`/`edges` |
| Flush | Événement `pagehide` — flush sync avant navigation |
| Clear | Quand `nodes.length === 0`, clé supprimée |
| Auth | **Non requise** |

**FACT :** La porte `hydratedRef` empêche l'autosave d'écraser les données restaurées pendant le fetch initial `?share=`.

**FACT :** Ordre mount : fetch `?share=<uuid>` d'abord → en cas d'échec fallback localStorage → puis activer autosave.

### 2 — URL Share (public, éphémère)

**Flow:**

```text
CanvasToolbar.handleShare
  → serializeCanvas(nodes, edges)
  → POST /api/share  (max 512KB, valide nœuds via isValidNode)
  → UUID stocké dans shared_canvases
  → URL: /canvas?share=<uuid>
Destinataire:
  useCanvasPersistence mount
  → GET /api/share/[id]
  → restoreCanvas(saved)
  → retirer ?share= de l'URL
```

**FACT :** Nettoyage TTL share : 5 % des POST suppriment lignes > 30 jours (`app/api/share/route.ts`).

**FACT :** Auth **non requise** pour créer ou ouvrir un lien share (rate-limité par IP).

### 3 — Workspaces sauvegardés (authentifié)

**Flow:**

```text
CanvasToolbar.handleSave (nécessite accessToken)
  → POST /api/workspace { name, data: SavedCanvas, nodeCount }
  → ligne saved_workspaces clé par userId

Chargement page Workspaces:
  → GET /api/workspace/[id]
  → appendWorkspace(saved)   // fusion, pas remplacement
  → router.push("/canvas")
```

**FACT :** `appendWorkspace` vs `restoreCanvas` :

| Fonction | Comportement |
| --- | --- |
| `restoreCanvas` | **Remplace** tout le canvas ; reset files UI ; sync `nodeIdCounter` |
| `appendWorkspace` | **Fusionne** graphe entrant ; remap ids en collision ; skip paires arêtes dupliquées ; `findFreeSlot` pour placement |

**FACT :** Merge invité à la connexion : `mergeGuestWorkspace(accessToken)` POST le canvas localStorage une fois, définit flag `open-hikmah-guest-merged` (`useCanvasPersistence.ts`).

### 4 — Paramètres URL deep-link (pas état canvas complet)

**Where:** `app/canvas/CanvasPageClient.tsx` → `VerseLoader`

| Param | Effet |
| --- | --- |
| `?verse=2:255` | Fetch API verset → `addVerseNode` → `setPendingAutoExpand` → nettoyer URL |
| `?surah=2` | Fetch tous ayahs → `addSurahNodes(missing)` → `requestFit` → nettoyer URL |

**FACT :** Ces params ajoutent du contenu au canvas **courant** (ou skip si ref déjà présente) — ils n'encodent pas le layout.

---

## Qu'est-ce qui survit au refresh ?

| Données | Survit au refresh navigateur ? | Mécanisme |
| --- | --- | --- |
| Nœuds/arêtes canvas | **Oui** (même navigateur) | Autosave localStorage |
| Viewport pan/zoom | **Non** | Pas dans `SavedCanvas` ; React Flow refit au chargement |
| Snapshot share | **Oui** (TTL serveur 30 jours) | UUID dans URL ou lien collé |
| Workspace sauvegardé | **Oui** (jusqu'à suppression) | Postgres + auth |
| Postgres `connections` | **Oui** (global) | Indépendant du canvas |
| Bookmarks | **Oui** | Persist `store/auth.ts` + sync API |
| Préférences UI (minimap) | **Oui** | `store/preferences.ts` → `open-hikmah-preferences` |

---

## Qu'est-ce qui nécessite l'authentification ?

| Fonctionnalité | Invité | Connecté |
| --- | --- | --- |
| Construire canvas localement | Oui | Oui |
| Expand (connexions IA) | Oui* | Oui* |
| Lien share | Oui | Oui |
| Sauver workspace | Non — « Sign in to save » | Oui |
| Lister/charger/supprimer workspaces | Non | Oui |
| Merge canvas invité → cloud | N/A | Une fois à la connexion |

\* Sous réserve des rate limits sur chemin miss IA — pas gated par auth.

---

## Mutation tracée : expand nœud → rendu

La boucle de mutation d'état canonique (connecte Phase 7 à Phase 8) :

```mermaid
sequenceDiagram
  participant U as Utilisateur
  participant VN as VerseNode
  participant Z as Zustand
  participant HC as HikmahCanvas
  participant API as POST /api/connections
  participant RF as ReactFlow

  U->>VN: Sélectionner "Theme" dans ExpandMenu
  VN->>Z: setPendingExpand({ nodeId, ref, kind })
  Z-->>HC: effet pendingExpand
  HC->>Z: getExpansionRefs(nodeId, kind)
  HC->>Z: setExpandingNode(nodeId)
  HC->>API: fromRef, kind, arabicText, translation, excludeRefs
  API-->>HC: ConnectionResult[]
  loop chaque connexion
    HC->>Z: addVerseNode(conn, pos) ou skip si hasNode
    HC->>Z: addConnectionEdge(buildConnectionEdge(...))
  end
  HC->>Z: setExpandingNode(null)
  Z-->>RF: nodes/edges mis à jour
  RF-->>U: nouveaux VerseNodes + HikmahEdges visibles
  Z-->>LS: sauvegarde debounced useCanvasPersistence
```

### Pas à pas

| Étape | Where | Changement d'état |
| --- | --- | --- |
| 1 | `VerseNode.handleExpandSelect` | `pendingExpand` défini |
| 2 | Effet `HikmahCanvas` | `pendingExpand` effacé ; `runExpansion` démarre |
| 3 | `runExpansion` | `expandingNodeId = nodeId` ; `excludeRefs` depuis arêtes |
| 4 | Réseau | `ConnectionResult[]` retourné |
| 5 | Par résultat | `radialPos` + `findFreeSlot` → `addVerseNode` **ou** arête seule si ref existe |
| 6 | Par résultat | `buildConnectionEdge` → `addConnectionEdge` |
| 7 | Finally | `expandingNodeId = null` ; `reactFlow.fitView` |
| 8 | Persistance | 800 ms plus tard → localStorage `serializeCanvas` |

**FACT :** `addVerseNode` définit aussi `newlyAddedNodeId` pour animation pulse et recalcule `duplicateNodeIdsByRef`.

**FACT :** `addConnectionEdge` recalcule `expansionCountsByNode` — les badges menu expand se mettent à jour immédiatement.

---

## Mutation tracée : recherche → premier nœud

| Étape | Where | Changement d'état |
| --- | --- | --- |
| 1 | `SearchDialog.selectResult` | Fetch `/api/verse/...` |
| 2 | Position | `viewportCenter` + `findFreeSlot` si canvas non vide |
| 3 | `addVerseNode({ ...verse, isRoot: isFirst }, position)` | Nouveau nœud à position calculée |
| 4 | Premier nœud uniquement | `setPendingAutoExpand(nodeId)` → expand thématique |
| 5 | Recherche sur canvas peuplé | `setPendingPanToNode(nodeId)` |
| 6 | Persistance | Autosave après debounce |

---

## Sidebar et présentation arête

**FACT :** Clic arête → `setSidebarContent({ type: "edge", fromVerse, toVerse, reason, kind, label })` → `ContextSidebar` affiche la raison IA avec **style éditorial** (pas style scripture — voir Phase 3 / `DESIGN.md`).

**FACT :** `HikmahEdge` colore par kind : variables CSS thematic / root / contrast ; le label est `reason` tronqué.

---

## Comparaison : ingénieur habitué à Firestore

| Si vous pensez… | Le canvas OpenHikmah en réalité… |
| --- | --- |
| Document = verset | **Instance nœud** = verset + position ; la même ref peut avoir plusieurs documents |
| Sync collection | **Pas de sync temps réel** — localStorage + save/share manuel optionnel |
| Layout autoritaire serveur | **Layout autoritaire client** ; le serveur stocke snapshots au share/save |
| Arêtes graphe en DB | **Deux couches** — Postgres `connections` (cache IA) vs arêtes canvas (session) |

---

## Tests clés

| Fichier | Ce qu'il verrouille |
| --- | --- |
| `__tests__/store/canvas.test.ts` | serialize/deserialize, getExpansionRefs, remap id appendWorkspace, dedupe |
| `__tests__/lib/canvas/canvas-layout.test.ts` | findFreeSlot, buildConnectionEdge |
| `__tests__/hooks/useCanvasPersistence.test.ts` | Restore share, merge invité, porte hydration |
| `__tests__/components/canvas/HikmahCanvas.test.tsx` | Boucle expansion, vide vs épuisé |
| `__tests__/app/canvas/CanvasPageClient.test.tsx` | Params URL `VerseLoader` |

---

## Modes d'échec

| Symptôme | Cause probable |
| --- | --- |
| Refresh perd le canvas | localStorage bloqué/quota ; ou effacé avant hydration |
| Share ouvre vide | UUID invalide/expiré ; fallback localStorage |
| Charger workspace empile bizarrement | `appendWorkspace` fusionne — attendu ; effacer d'abord si remplacement voulu |
| Nœuds dupliqués même ref | Autorisé par design ; indicateur affiché sur nœud |
| Viewport saute au reload | Viewport non persisté ; `fitView` s'exécute sur premiers nœuds |
| Bouton Save absent | Non connecté — invité utilise localStorage uniquement |

---

## À COMPRENDRE MAINTENANT

1. **L'état canvas vit dans Zustand** — React Flow est une vue contrôlée.
2. **Id nœud ≠ ref verset** — les refs peuvent se répéter ; les ids sont générés côté client.
3. **`SavedCanvas` sauve layout + snapshots** — pas graphe Postgres, pas viewport, pas files UI.
4. **`restoreCanvas` remplace ; `appendWorkspace` fusionne** — sémantiques différentes.
5. **L'expansion écrit dans Zustand** après retour API — le pipeline Phase 7 se termine dans `addVerseNode` / `addConnectionEdge`.
6. **localStorage fonctionne sans auth** — workspaces et merge nécessitent connexion.

---

## UTILE PLUS TARD

- `lib/canvas/canvas-export.ts` — export PNG/PDF depuis toolbar (nœuds uniquement, pas arêtes sur chemin PNG).
- `hooks/useActivityTracker.ts` — signaux streak/social ; utilise `restoreToken` pour ignorer restores bulk.
- `components/home/PersonalHome.tsx` — « continue where you left off » lit `CANVAS_STORAGE_KEY`.
- Retrait arête dans admin Postgres — ne retire pas automatiquement les arêtes canvas.

---

## IGNORER POUR L'INSTANT

- Ordre lecture graphe audio dans toolbar (`playGraph`) — orthogonal à persistance graphe.
- Overlay onboarding `CanvasTour` — UX uniquement.
- Génération image OG pour shares — aperçu social, pas chemin restore.

---

## Questions checkpoint Phase 8

1. Quelles trois couches de stockage contiennent des données « graphe-like », et en quoi diffèrent-elles ?
2. Pourquoi la même `verse.ref` peut apparaître sur deux nœuds, et que vérifie `hasNode` ?
3. Quels champs sont dans `SavedCanvas`, et qu'est-ce qui est délibérément omis ?
4. Quelle est la différence entre `restoreCanvas` et `appendWorkspace` ?
5. Tracez `getExpansionRefs` — quelles arêtes comptent pour « get more » ?

---

**Suivant :** [Phase 9 — Base de données et recherche sémantique](./phase-9-database-and-semantic-search.md) — schéma Postgres, Drizzle, pgvector, seeding et tests d'intégration en profondeur ingénieur.

Dites **« continue to Phase 9 »** quand vous êtes prêt, ou demandez un zoom sur n'importe quel sujet canvas/état.

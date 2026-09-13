# Phase 4 — Modèle mental d'architecture

> **Prérequis :** [Phase 1](./phase-1-what-is-openhikmah.md) · [Phase 2](./phase-2-domain-primer.md) · [Phase 3](./phase-3-theological-boundaries.md)  
> **Tags de preuve :** **FACT** · **INFERENCE** · **UNKNOWN**

La Phase 4 reconstruit **comment le système est réellement structuré** — pas la théorie générale de Next.js. Quand ce dépôt diffère des anciennes suppositions Next, ces différences sont indiquées.

---

## Commencez ici : un parcours utilisateur dans l'architecture

Imaginez que vous cherchez `2:255`, le placez sur le canvas, étendez par **Thème**, et lisez trois versets connectés.

```mermaid
flowchart TB
  subgraph browser ["Navigateur (client)"]
    UI["Pages React + composants"]
    ZC["Zustand: canvas, auth, prefs, social, audio"]
    LS[("localStorage: sauvegarde auto canvas")]
    UI --> ZC
    ZC --> LS
  end

  subgraph edge ["Next.js 16 App Router"]
    PAGES["app/**/page.tsx — RSC + îlots client"]
    API["app/api/**/route.ts — Route Handlers"]
    PROXY["proxy.ts — nonce CSP, maintenance"]
  end

  subgraph lib ["lib/ — logique domaine"]
    Q["lib/quran/* — corpus, recherche, morphologie"]
    AI["lib/ai/* — graph-service, generator, embed"]
    CV["lib/canvas/* — mise en page, validation partage"]
    AU["lib/auth/* — PKCE, vérif JWT, requireUser"]
  end

  subgraph data ["Persistance et externes"]
    PG[("PostgreSQL + pgvector")]
    RD[("Redis — cache optionnel")]
    EXT["alquran.cloud · quran.com · Claude/Gemini"]
  end

  UI --> PAGES
  UI -->|"fetch JSON"| API
  PROXY --> PAGES
  API --> lib
  lib --> PG
  lib --> RD
  lib --> EXT
  API -->|"ConnectionResult[]"| UI
```

**À COMPRENDRE MAINTENANT :** Le navigateur **explore** ; le serveur **valide, génère (sur miss), et persiste** la vérité partagée. Votre mise en page canvas est surtout à vous jusqu'à ce que vous partagiez ou sauvegardiez un workspace.

---

## Aperçu de la stack (vérifié)

**FACT :** Depuis `README.md` et `package.json` :

| Couche | Choix |
| --- | --- |
| Framework | Next.js **16.2** App Router, React **19**, TypeScript strict |
| i18n | next-intl (`i18n/request.ts`, `messages/*.json`) |
| UI Canvas | `@xyflow/react` |
| État client | Zustand (+ `persist` pour favoris auth, préférences) |
| Style | Tailwind CSS v4 |
| IA | Anthropic Claude (+ secours Gemini) ; *embeddings* Gemini |
| DB | PostgreSQL + pgvector + Drizzle ORM |
| Cache optionnel | Redis (`lib/infra/redis.ts`) |
| Auth | Quran Foundation OAuth2 PKCE |
| Tests | Vitest (unit/integration), Playwright (e2e) |
| Gestionnaire de paquets | Bun |

**FACT :** `next.config.ts` met `typescript.ignoreBuildErrors: true` pour le build production **OOM** — **la CI exécute toujours `bun run typecheck`** comme barrière dure. Ne traitez pas le succès du build comme la sécurité des types.

**FACT :** Ce dépôt utilise **`proxy.ts`**, pas `middleware.ts`, pour les nonces CSP par requête et le mode maintenance (`proxy.ts` commentaires d'en-tête référencent l'ordre de routage Next 16).

---

## Frontière client vs serveur

### Server Components (par défaut)

**FACT :** Le `app/layout.tsx` racine est un Server Component — charge la locale via `getUiLocale()`, les messages via `getMessages()`, les passe au client `Providers`.

**FACT :** Beaucoup de pages sont des enveloppes serveur fines + îlots client, ex. `app/canvas/page.tsx` → `CanvasPageClient` (`"use client"`).

**INFERENCE :** Les Server Components sont utilisés où SEO, cookies, et locale initiale comptent ; l'interactivité lourde va aux enfants client.

### Îlots client seulement

**FACT :** `HikmahCanvas` est chargé avec `dynamic(..., { ssr: false })` — le canvas React Flow ne fait pas de SSR (`CanvasPageClient.tsx`).

**FACT :** Tous les stores Zustand sont des modules client (`"use client"` dans les fichiers store ou consommés seulement depuis composants client).

### Route Handlers API = backend

**FACT :** `app/api/**/route.ts` exporte `GET`/`POST`/etc. — c'est le **backend applicatif** pour la SPA. Pas de serveur Express séparé.

**INFERENCE :** Comparaison Firebase : Route Handlers ≈ endpoints HTTP Cloud Functions ; `lib/` ≈ logique de fonction partagée ; Drizzle + Postgres ≈ couche Firestore/SQL (mais relationnelle + pgvector).

---

## Référence des couches (carte des responsabilités)

Pour chaque couche : **ce qu'elle possède**, **qui l'appelle**, **invariants critiques**.

### 1. Pages et routage (`app/`)

| | |
| --- | --- |
| **Responsabilité** | URL → enveloppe UI ; métadonnées ; préchargement données serveur quand c'est peu coûteux |
| **Entrées** | URL requête, cookies (`oh_locale`, `oh_edition`), searchParams |
| **Sorties** | HTML + arbre composants client |
| **État possédé** | Aucun de longue durée (par requête) |
| **Appelle** | `lib/quran/*`, `lib/i18n/request-prefs`, composants client |
| **Appelé par** | Navigation navigateur |

**Routes clés (produit central) :**

| Chemin | Rôle |
| --- | --- |
| `/` | Page d'accueil + Verset du jour |
| `/canvas` | Espace de graphe infini |
| `/search` | Page de recherche complète |
| `/names`, `/names/[slug]` | 99 Noms divins |
| `/stories/[slug]` | Récits prophétiques (données statiques) |
| `/bookmarks`, `/workspaces`, `/social` | Fonctionnalités authentifiées |
| `/admin/*` | Console opérateur (liste blanche env) |
| `/callback` | Fin OAuth PKCE |

**Invariant :** Les liens profonds comme `/canvas?verse=2:255` ou `?surah=18` s'hydratent côté client via fetch API (`CanvasPageClient` `VerseLoader`).

---

### 2. Composants UI (`components/`)

| | |
| --- | --- |
| **Responsabilité** | Présentation, interaction, accessibilité |
| **Entrées** | Props, sélecteurs Zustand, réponses fetch |
| **Sorties** | Événements → mises à jour store / appels API |
| **État possédé** | UI éphémère seulement (dialogues ouverts, survol) |
| **Appelle** | `store/*`, `fetch('/api/...')` |
| **Appelé par** | Pages `app/*` |

**Clusters principaux :**

| Dossier | Rôle |
| --- | --- |
| `components/canvas/` | `HikmahCanvas`, `VerseNode`, `ExpandMenu`, barre d'outils, export |
| `components/search/` | `SearchDialog`, résultats sourate |
| `components/layout/` | En-tête, barre latérale, enveloppe auth |
| `components/admin/` | Backfill, couverture, prompts (haut risque) |
| `components/audio/` | Mini lecteur pour récitation |

**Invariant (`DESIGN.md`) :** UI d'explication IA ≠ style verset.

---

### 3. État client (`store/` + `hooks/`)

| Store | Persiste ? | Possède |
| --- | --- | --- |
| `store/canvas.ts` | Via hook → localStorage | nodes, edges, sélection, expand pending, viewport |
| `store/auth.ts` | Favoris dans localStorage ; token **mémoire seulement** | accessToken, état sync favoris |
| `store/preferences.ts` | localStorage + cookies | locale, édition par locale, récitateur, prefs canvas |
| `store/social.ts` | Persist partiel | profil, série, compteurs pending |
| `store/audio.ts` | Session | file lecture, verset courant |

| Hook | Rôle |
| --- | --- |
| `hooks/useCanvasPersistence.ts` | Restaure URL partage → autosave localStorage ; `buildShareUrl` ; fusion invité→workspace |
| `hooks/useActivityTracker.ts` | Poste activité sociale sur actions canvas |

**Niveaux de persistance canvas (FACT depuis `useCanvasPersistence.ts`) :**

1. **En mémoire** — Zustand (session live)
2. **localStorage** — autosave `open-hikmah-canvas` avec debounce
3. **URL de partage** — `POST /api/share` → UUID → `/canvas?share=<uuid>` → `GET /api/share/[id]`
4. **Workspace** — `POST /api/workspace` authentifié (Postgres `saved_workspaces`)

**Invariant :** La restauration partage s'exécute **avant** la restauration localStorage ; l'autosave attend la fin de l'hydratation (évite d'effacer le canvas partagé).

---

### 4. Routes API (`app/api/`)

Groupées par domaine :

| Préfixe | Responsabilité | Auth |
| --- | --- | --- |
| `/api/search` | Mot-clé + sémantique liée | Public (limité en débit) |
| `/api/verse/...`, `/api/verses/...` | Recherche verset, similaire, tafsir | Surtout public |
| `/api/connections` | **Expansion graphe** — cache + miss IA | Public (limité en débit sur chemin IA) |
| `/api/share` | Snapshot canvas persisté | Public (limité en débit) |
| `/api/workspace` | Canvas sauvegardés nommés | Bearer token |
| `/api/bookmarks`, `/api/notes` | Données verset utilisateur | Bearer token |
| `/api/auth/*` | Échange PKCE, refresh, déconnexion | Cookie + token |
| `/api/social/*` | Amis, défis, séries, mentions | Bearer token |
| `/api/names/...` | Contenu IA noms (versets, réflexion, associations) | Public (limité en débit) |
| `/api/admin/*` | Prompts, backfill, flags, audit | Liste blanche admin |
| `/api/health`, `/api/metrics` | Ops | Public / interne |

**Frontière de confiance (FACT) :** Les routes appellent `isValidRef`, `requireUser`, `requireAdmin` **avant** la logique domaine — voir Phase 3.

---

### 5. Bibliothèques domaine (`lib/`)

#### `lib/quran/` — texte canonique et récupération

| Module | Rôle |
| --- | --- |
| `quran-corpus.ts` | Lectures `verses` locales, `isValidRef` |
| `verse-resolver.ts` | Corpus + secours alquran.cloud |
| `semantic-search.ts` | Requêtes pgvector, cache embed requête |
| `arabic-morphology.ts` | Aides surlignage tokens |
| `chapters.ts`, `surah-names.ts` | Métadonnées sourate |
| `audio.ts` | URLs récitation / ordre |

**Appelle :** Drizzle → Postgres ; *embed* Gemini (requêtes) ; Redis optionnel.  
**Appelé par :** Routes API, `lib/ai/graph-service`, scripts.

#### `lib/ai/` — probabiliste + orchestration graphe

| Module | Rôle |
| --- | --- |
| `graph-service.ts` | **Graphe persistant** — lecture cache, génération miss, traduction locale |
| `connection-discovery.ts` | Candidats déterministes (racines / vecteurs) |
| `connection-generator.ts` | **Seul appelant LLM pour connexions** |
| `ai.ts` | Résolution fournisseur, `callAI`, `embed` |
| `translate.ts` | Traduction raison localisée + validation |
| `connection-batch*.ts` | Backfill admin |
| `prompt-registry.ts` | Versions prompt DB avec fallback codé en dur |
| `theological-constraints.ts` | `TANZIH_CONSTRAINT` partagé |

**Invariant :** Sélection de versets canonique anglaise ; les locales traduisent les **raisons** seulement.

#### `lib/canvas/` — mise en page et validation partage

| Module | Rôle |
| --- | --- |
| `canvas-layout.ts` | Placement nœuds sans collision |
| `share-canvas.ts` | `isValidNode` pour POST partage |
| `canvas-export.ts` | Export PDF/image |

#### `lib/auth/` — identité

| Module | Rôle |
| --- | --- |
| `pkce.ts` | Construit URL autorisation QF (aides côté client possibles) |
| `social-auth.ts` | Vérif JWT (JWKS), `requireUser`, cache token |
| `session-cookie.ts` | Constantes cookie refresh HttpOnly |

**FACT :** Token d'accès en **mémoire** ; refresh via cookie HttpOnly (`store/auth.ts` commentaire).

#### `lib/infra/` — transversal

| Module | Rôle |
| --- | --- |
| `db.ts` + `db/schema.ts` | Client Drizzle + toutes les tables |
| `rate-limit.ts` | Fenêtres de débit Postgres et/ou Redis |
| `redis.ts` | Cache optionnel (requêtes embed, auth L2, limites débit) |
| `http.ts` | `clientKey` depuis IP transmise |
| `metrics.ts` | Compteurs ops |
| `search-log.ts` | Analytics recherche anonymes |

#### `lib/names/`, `lib/stories/`, `lib/social/`

- **Noms :** Données noms divins statiques + cache contenu IA (`name-content.ts`)
- **Récits :** Récits TS statiques (`lib/stories/data/*.ts`) — pas d'IA au runtime
- **Social :** Logique séries, amis, défis, publication activité

#### `lib/admin/`

Feature flags, exécuteur de jobs, rapports couverture, auth admin — géré par **CODEOWNERS**.

---

### 6. PostgreSQL (via Drizzle)

**FACT :** Un seul `DATABASE_URL` ; schéma Drizzle dans `lib/infra/db/schema.ts`.

**Tables produit principales :**

| Table | But |
| --- | --- |
| `verses`, `verse_translations` | Texte coranique canonique |
| `word_morphology` | Découverte racines |
| `verse_embeddings` | Recherche sémantique + candidats thématiques |
| `connections` | **Liens du graphe de connaissances partagé** |
| `connection_coverage` | Comptabilité backfill admin |
| `shared_canvases` | Snapshots lien de partage |
| `saved_workspaces` | Canvas nommés utilisateur |
| `users`, `bookmarks`, `verse_notes`, … | Auth + social |
| `ai_generations`, `prompt_versions` | Audit + admin prompts |
| `name_content`, `name_verse_reasons` | Cache IA Noms divins |

**Invariant :** Les liens du graphe sont **globaux** — une génération sert tous les utilisateurs (coût → 0 avec le temps).

---

### 7. Services externes

| Service | Utilisé pour |
| --- | --- |
| **alquran.cloud** | Seed corpus ; secours verset en direct |
| **API quran.com** | Index recherche mot-clé ; localisation noms sourate ; recherche versets noms |
| **Anthropic / Gemini** | Raisons connexions, contenu noms, traductions |
| ***Embeddings* Gemini** | Recherche sémantique (toujours Gemini) |
| **OAuth Quran Foundation** | Connexion, claims JWT |
| **Google Analytics** | Usage (script avec nonce via CSP) |

---

## Flux de requêtes (niveau architecture)

### Flux A — Étendre un verset sur le canvas

```
VerseNode ExpandMenu
  → HikmahCanvas.runExpansion()
  → POST /api/connections { fromRef, kind, arabicText, translation, excludeRefs }
  → getConnections() [graph-service]
       → SELECT connections (cache hit?) → return
       → discoverCandidates() → generateGroundedConnections() → INSERT connections
  → ConnectionResult[] JSON
  → canvas store addVerseNode + addConnectionEdge
  → useCanvasPersistence debounced localStorage save
```

**Flèche de confiance :** Le LLM est **en aval de** la découverte + validation (Phase 2).

### Flux B — Recherche

```
SearchDialog → GET /api/search?q=...
  → chemin ref | nom sourate | mot-clé (quran.com) + sémantique liée optionnelle
  → hydrate via getVerses (corpus local)
  → SearchResponse JSON
```

Les budgets mot-clé et sémantique sont des **clés de limite de débit séparées** (`searchkw:` vs `search:`).

### Flux C — Partager le canvas

```
CanvasToolbar → serializeCanvas() → POST /api/share
  → ligne shared_canvases (UUID)
  → URL /canvas?share=<uuid>
Destinataire ouvre URL → GET /api/share/[id] → restoreCanvas()
```

Mise en page seulement — **pas** le cache global `connections`.

### Flux D — Connexion + sync favoris

```
PKCE → /callback → /api/auth/exchange
  → token d'accès (mémoire) + cookie refresh
  → SessionRestorer → /api/auth/refresh au chargement
  → loadRemoteBookmarks → fusion refs pending locales
  → mergeGuestWorkspace (canvas local → workspace cloud une fois)
```

---

## Architecture de localisation

**FACT :** Deux canaux de préférence parallèles :

| Canal | Stockage | Consommé par |
| --- | --- | --- |
| Locale UI | Cookie `oh_locale` (+ miroir localStorage) | Messages next-intl |
| Édition Coran | Cookie `oh_edition` | Hydratation verset/recherche API serveur |

**FACT :** `getUiLocale()` / `getQuranEdition()` valident contre listes blanches (`lib/i18n/request-prefs.ts`) — cookies invalides retombent en sécurité.

**FACT :** Les **raisons** de connexion sont servies dans la locale UI quand en cache ; la **sélection** de versets reste canonique anglaise (Phase 2/3).

---

## Audio (périphérique mais réel)

**FACT :** `store/audio.ts` + `components/audio/MiniPlayer.tsx` — lit les versets du canvas dans l'ordre du Coran (fonctionnalité `README`).

**UTILE PLUS TARD :** Préférence récitateur dans `store/preferences.ts` (`DEFAULT_RECITER` depuis `lib/quran/audio.ts`).

---

## Social / défis (périphérique)

**FACT :** Tables : `friendships`, `challenges`, `activity_log`, `note_mentions`, etc.

**FACT :** `hooks/useActivityTracker.ts` poste l'activité quand l'utilisateur ajoute versets/connexions — alimente séries et défis.

**IGNORER POUR L'INSTANT :** Cycle de vie des défis, score classement — sauf contribution dans cette zone.

---

## Admin et opérations

**FACT :** Admin = liste blanche env `ADMIN_QF_IDS` — échec fermé (`lib/admin/admin-auth.ts`).

**FACT :** Le mode maintenance de `proxy.ts` lit le feature flag `maintenance_mode` — exclut admin/auth/health du 503.

**FACT :** Scripts (`scripts/*.mjs`, `scripts/*.ts`) : seed Coran, morphologie, embeddings, migrate, backfill, prewarm.

**FACT :** Le hook pre-push exécute les tests d'intégration (Testcontainers) — nécessite Docker.

---

## Architecture des tests (où vit la preuve)

| Couche | Emplacement | Ce qui s'exécute |
| --- | --- | --- |
| Unit | `__tests__/lib/`, `__tests__/api/`, `__tests__/store/` | Vitest, fetch/IA mockés |
| Integration | `__tests__/integration/` | Postgres réel via Testcontainers |
| E2E | `e2e/` | Playwright + axe |
| CI | `.github/workflows/ci.yml` | lint, typecheck, unit, integration, build, e2e |

**INFERENCE :** Les tests unitaires prouvent les contrats de modules ; les tests d'intégration prouvent DB + persistance graphe ; les e2e prouvent les flux utilisateur.

---

## Mermaid — frontières de confiance

```mermaid
flowchart LR
  subgraph trusted ["Déterministe / validé serveur"]
    REF[isValidRef]
    CORP[Corpus Postgres]
    DISC[discoverCandidates]
  end

  subgraph probabilistic ["LLM (serveur seulement)"]
    LLM[connection-generator / IA noms]
  end

  subgraph client ["Client (entrée non fiable)"]
    BR[UI canvas + recherche navigateur]
  end

  BR -->|"requêtes JSON"| API["Routes API"]
  API --> REF
  REF --> CORP
  REF --> DISC
  DISC --> LLM
  LLM -->|"sortie validée"| CORP
  CORP -->|"réponses JSON"| BR
```

Le client peut **suggérer** des refs et du texte dans les corps POST ; le serveur **re-valide** avant l'IA et la persistance.

---

## Cinq faits d'architecture à mémoriser

1. **Trois niveaux de persistance pour l'exploration :** canvas en mémoire → localStorage → (optionnel) URL partage / workspace — **séparé de** le graphe global `connections` dans Postgres.

2. **`lib/ai/graph-service.ts` est le cerveau du graphe :** cache d'abord, IA sur miss, liens partagés écrits une fois — l'expansion canvas passe toujours par `/api/connections`.

3. **Tous les appels LLM pour connexions de versets vivent dans `lib/ai/connection-generator.ts`**, invoqués seulement depuis pipelines graphe/noms côté serveur — pas depuis composants React directement.

4. **Les tokens client restent en mémoire ; le refresh utilise des cookies HttpOnly** — les favoris persistent localement pour les invités ; les utilisateurs connectés synchronisent via `/api/bookmarks`.

5. **`proxy.ts` (pas `middleware.ts`) + Route Handlers + Drizzle** définissent la forme Next 16 de ce dépôt — ne supposez pas Pages Router ou accès DB côté client.

---

## Différences Next.js à noter

| Ancienne supposition | Réalité OpenHikmah |
| --- | --- |
| `middleware.ts` | **`proxy.ts`** avec `config.matcher` |
| Build = types sûrs | **`ignoreBuildErrors: true`** dans build prod ; la CI typecheck séparément |
| SSR canvas | **`ssr: false`** pour React Flow |
| API dans `pages/api` | **`app/api/**/route.ts`** |
| Env lu n'importe où | Secrets serveur seulement dans Route Handlers / `lib/` — `NEXT_PUBLIC_*` figé au démarrage dev |

**UNKNOWN pour vous jusqu'à la Phase 10 :** Bootstrap dev local exact (Docker compose, migrate, ordre seed) — documenté dans CONTRIBUTING / Phase 10.

---

## Phase 4 — Ce qu'il faut retenir

### À COMPRENDRE MAINTENANT

- **`app/`** = routes ; **`components/`** = UI ; **`lib/`** = domaine ; **`store/`** = état session client ; **`app/api/`** = backend.
- **État canvas ≠ graphe de connexions** — mise en page locale/partageable ; liens canoniques dans Postgres.
- **Chemin expansion :** client → `/api/connections` → `graph-service` → (peut-être) IA → DB → JSON → Zustand.
- **Auth :** PKCE + JWT ; admin via liste blanche env ; chemins à haut risque gérés par CODEOWNERS.
- **Postgres tient la vérité sacrée + graphe ; Redis est une accélération optionnelle.**

### UTILE PLUS TARD

- Architecture job backfill admin (`connection-batch-loop.ts`)
- Bascule application CSP depuis report-only
- Idempotence flag `mergeGuestWorkspace`
- File activité sociale (`post-activity.ts`)

### IGNORER POUR L'INSTANT

- Routes génération image OG
- Réglage endpoint rapport CSP
- Détails job CI size-limit

---

## Incertitudes

| Sujet | Statut |
| --- | --- |
| Single-flight multi-instance pour misses graphe | **FACT :** en processus seulement ; verrou Redis reporté (`graph-service.ts` commentaire) |
| Toutes les pages utilisent RSC vs client | **INFERENCE :** pattern hybride ; canvas/recherche fortement client |
| Hébergement production (Coolify mentionné dans next.config) | **FACT :** sortie standalone ; topologie déploiement complète **UNKNOWN** sans docs ops |

---

## Et ensuite

**Phase 5 — Visite du dépôt :** carte de répertoires ciblée — ce qui va où, quand vous le touchez, niveau de risque pour nouveaux contributeurs (pas un dump d'arborescence complet).

Dites **« continue vers la Phase 5 »** quand vous êtes prêt.

> **Version complète (français B2+) :** [Phase 4](../onboarding-fr/phase-4-architecture.md)

---

## Fichiers clés (liste de lecture Phase 4)

| Fichier | Pourquoi |
| --- | --- |
| `app/canvas/CanvasPageClient.tsx` | Séparation client/serveur, liens profonds |
| `components/canvas/HikmahCanvas.tsx` | Expansion → API → store |
| `hooks/useCanvasPersistence.ts` | localStorage + restauration partage |
| `app/api/connections/route.ts` | Frontière API graphe |
| `lib/ai/graph-service.ts` | Orchestration cache + génération |
| `lib/infra/db/schema.ts` | Propriété des tables |
| `store/canvas.ts` | Modèle état canvas |
| `store/auth.ts` | Modèle token + favoris |
| `proxy.ts` | Bord requête (CSP, maintenance) |
| `components/providers.tsx` | Restauration session, sync locale |
| `lib/auth/social-auth.ts` | Frontière `requireUser` |
| `lib/ai/ai.ts` | Résolution fournisseur/fonctionnalité |

# Phase 10 — Lancer l'application localement

> **Prérequis :** Phases [1](../onboarding-fr/phase-1-what-is-openhikmah.md)–[9](./phase-9-database-and-semantic-search.md)  
> **Étiquettes de preuve :** **FACT** · **INFERENCE** · **UNKNOWN**

La phase 9 expliquait **ce que Postgres stocke**. La phase 10 est **pratique** — comment lancer OpenHikmah sur votre machine, quelles variables d'environnement activent quelles fonctions, et comment voir si tout marche.

**Cette phase documente le workflow.** Lancez les commandes vous-même quand vous êtes prêt ; rien ici ne modifie le code de l'application.

---

## Ce que vous construisez en local

```mermaid
flowchart LR
  subgraph machine ["Your machine"]
    Bun[bun run dev :3000]
    Browser[Browser]
    LS[(localStorage canvas)]
  end

  subgraph services ["Local services"]
    PG[(Postgres + pgvector)]
    Redis[(Redis — optional)]
  end

  subgraph external ["External APIs"]
    Claude[Anthropic Claude]
    Gemini[Gemini embed + optional LLM]
    QF[Quran Foundation OAuth]
    QC[quran.com search]
    AQ[alquran.cloud fallback]
  end

  Browser --> Bun
  Bun --> PG
  Bun -.-> Redis
  Bun --> Claude
  Bun --> Gemini
  Bun --> QF
  Bun --> QC
  Bun --> AQ
  Browser --> LS
```

**À COMPRENDRE MAINTENANT :** OpenHikmah n'est **pas** un site statique. La recherche, l'expand, le partage, l'auth et le graphe IA ont besoin des **routes serveur + Postgres** pour un comportement complet.

---

## Pourquoi Bun ?

**FACT:** `package.json` déclare `"packageManager": "bun@1"`. CI et le Dockerfile utilisent `oven-sh/setup-bun` / `oven/bun:1-alpine`.

| Raison | Détail |
| --- | --- |
| Outil officiel | `bun install`, `bun run dev`, scripts via `bun scripts/...` |
| Vitesse | Installations rapides — utile sur un gros arbre de dépendances Next.js |
| Lockfile | `bun.lock` — CI utilise `bun install --frozen-lockfile` |

**FACT:** Vous n'avez pas besoin de Node/npm au quotidien si Bun est installé. Playwright e2e utilise toujours `bunx playwright install`.

**UNKNOWN:** Si les mainteneurs testent une version patch précise de Bun au-delà de `@1` — CI utilise `latest`.

---

## Guide pas à pas (étapes ordonnées)

Chaque étape : **quoi**, **changement d'état**, **succès**, **échec courant**.

### Étape 1 — Cloner et installer les dépendances

1. Allez dans votre dossier du projet.
2. Lancez `bun install`.

```bash
cd openhikmah-web    # votre clone fork
bun install
```

| | |
| --- | --- |
| **Quoi** | Installe les paquets depuis `bun.lock` dans `node_modules/` |
| **Changement d'état** | Disque uniquement — pas de `.env`, pas de base de données |
| **Succès** | La commande se termine avec le code 0 ; `node_modules/` rempli |
| **Échec** | Bun manquant → installer depuis [bun.sh](https://bun.sh) ; conflit de lockfile → ne pas supprimer le lockfile sans guidance des mainteneurs |

---

### Étape 2 — Créer `.env.local`

1. Copiez le fichier d'exemple.
2. Vérifiez que `.env.local` existe à côté de `.env.example`.

```bash
cp .env.example .env.local
```

| | |
| --- | --- |
| **Quoi** | Next.js charge `.env.local` au démarrage du serveur de dev (non versionné) |
| **Changement d'état** | Nouveau fichier gitignored |
| **Succès** | Fichier existant à côté de `.env.example` |
| **Échec** | Ne jamais committer `.env.local` — contient des secrets |

**FACT:** Après modification de `.env.local`, **redémarrez `bun run dev`**. Les variables `NEXT_PUBLIC_*` sont intégrées au bundle client au démarrage (ligne 129 de `.env.example`).

---

### Étape 3 — Démarrer Postgres avec pgvector

OpenHikmah requiert **PostgreSQL 16 + pgvector** (extension pour les vecteurs) (`docker-compose.yml`, migration `0008`).

**Option A — Docker Compose (app + db + redis) :**

1. Définissez `DB_PASSWORD` dans `.env` ou le shell.
2. Lancez la base de données.

```bash
# Set DB_PASSWORD in .env or shell, then:
docker compose up -d db
# Optionally: docker compose up -d   # full stack
```

**Option B — Postgres uniquement (correspond à l'URL de `.env.example`) :**

1. Lancez un conteneur Docker avec l'image pgvector.
2. Vérifiez avec `docker ps` que le conteneur tourne.

```bash
docker run -d --name openhikmah-db \
  -e POSTGRES_DB=open_hikmah \
  -e POSTGRES_USER=openh \
  -e POSTGRES_PASSWORD=devpassword \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

3. Ajoutez l'URL dans `.env.local` :

```bash
DATABASE_URL=postgresql://openh:devpassword@localhost:5432/open_hikmah
```

| | |
| --- | --- |
| **Quoi** | Instance Postgres vide avec extension vector disponible |
| **Changement d'état** | Volume Docker `postgres_data` (compose) ou système de fichiers du conteneur |
| **Succès** | `docker ps` montre le conteneur healthy ; port 5432 accessible |
| **Échec** | Port occupé → arrêter l'autre Postgres ou changer le mapping de port ; mauvaise image → doit être **pgvector**, pas `postgres:16` vanilla |

**INFERENCE:** `.env.example` référence une commande docker run du README qui peut ne pas apparaître dans le README actuel — l'option B ci-dessus correspond aux identifiants `DATABASE_URL` documentés.

---

### Étape 4 — Appliquer les migrations

1. Lancez les migrations principales.
2. Lancez les index concurrents (une fois en local).

```bash
bun run db:migrate
bun run db:migrate:concurrent-indexes   # connections locale index — run once locally
```

| | |
| --- | --- |
| **Quoi** | Crée toutes les tables depuis `lib/infra/db/migrations/` |
| **Changement d'état** | Schéma Postgres (tables vides) |
| **Succès** | `Migrations applied` / pas d'erreur |
| **Échec** | `DATABASE_URL` incorrect → connexion refusée ; pgvector manquant → la migration `0008` échoue |

Wrapper : `scripts/migrate.mjs` utilise le migrator Drizzle. Docker production exécute aussi `scripts/ensure-tables.mjs` comme bootstrap idempotent — optionnel en local après migrate.

---

### Étape 5 — Charger le corpus du Coran (requis pour un dev sérieux)

1. Lancez le script de seed du Coran.
2. Attendez le message de succès (~6236 versets).

```bash
bun run scripts/seed-quran.mjs
# or: node scripts/seed-quran.mjs  (scripts are .mjs; package.json has bun wrappers for some)
```

| | |
| --- | --- |
| **Quoi** | Récupère alquran.cloud (Arabe Uthmani + `en.sahih`) ; upsert ~6236 lignes dans `verses` |
| **Changement d'état** | Table `verses` remplie |
| **Succès** | Le log indique 6236 versets fusionnés (avertit si le compte diffère) |
| **Échec** | Pas de réseau → échec du fetch ; pas de `DATABASE_URL` → sortie immédiate |

**FACT:** Sans chargement, `getVerses()` manque et `resolveVerse()` **retombe sur alquran.cloud live** (`lib/quran/verse-resolver.ts`) — utilisable pour l'affichage basique de versets mais plus lent et pas le comportement CI/tests d'intégration.

---

### Étape 6 — Charger les données d'ancrage (fortement recommandé)

1. Chargez la morphologie des mots.
2. Lancez l'embed du corpus (nécessite `GEMINI_API_KEY`).

```bash
bun run scripts/seed-morphology.mjs   # word_morphology from data/morphology/*.jsonl
bun run embed                         # alias: scripts/embed-corpus.mjs — needs GEMINI_API_KEY
```

| Script | Débloque |
| --- | --- |
| `seed-morphology.mjs` | Découverte ancrée **Root** (chemin racine de `discoverCandidates`) |
| `embed-corpus.mjs` | Découverte **thematic/contrast** + recherche sémantique + « related by meaning » |

| | |
| --- | --- |
| **Quoi** | Tables d'ancrage niveau 1 (phase 9) |
| **Changement d'état** | `word_morphology`, `verse_embeddings` remplies |
| **Succès** | Morphologie log par fichier ; embed log `verse_embeddings table now holds N rows` |
| **Échec** | Embed sans `GEMINI_API_KEY` → script sort ; rate limit 429 → relancer plus tard (idempotent) |

**FACT:** `bun run embed` est défini dans `package.json` comme `bun scripts/embed-corpus.mjs`.

---

### Étape 7 — Définir les clés IA (minimum pour expand)

1. Ouvrez `.env.local`.
2. Ajoutez au minimum une clé pour l'expand IA.

Dans `.env.local`, au minimum pour **l'expand de connexion IA** :

```bash
ANTHROPIC_API_KEY=sk-ant-...
AI_PROVIDER=claude
```

**Alternative économique en dev :**

```bash
AI_PROVIDER=gemini
GEMINI_API_KEY=...
```

**FACT:** `AGENTS.md` / `CONTRIBUTING.md` : *« Au minimum vous avez besoin de `ANTHROPIC_API_KEY` pour tester les connexions IA. »* Gemini fonctionne quand `AI_PROVIDER=gemini`.

**Pour la recherche sémantique / embed corpus :**

```bash
GEMINI_API_KEY=...   # required even when AI_PROVIDER=claude
```

Les embeddings sont **toujours Gemini** (`lib/ai/ai.ts`).

---

### Étape 8 — Démarrer le serveur de dev

1. Lancez le serveur.
2. Ouvrez [http://localhost:3000](http://localhost:3000) dans le navigateur.

```bash
bun run dev
```

| | |
| --- | --- |
| **Quoi** | `next dev` — serveur de dev Next.js 16 App Router |
| **Changement d'état** | Processus à l'écoute sur le port 3000 (par défaut) |
| **Succès** | Le terminal affiche ready ; [http://localhost:3000](http://localhost:3000) charge |
| **Échec** | Port 3000 occupé → `bun run dev -- -p 3001` et mettre à jour `NEXT_PUBLIC_APP_URL` ; dépendances manquantes → relancer `bun install` |

**FACT:** Définissez `NEXT_PUBLIC_APP_URL=http://localhost:3000` dans `.env.local` pour la cohérence des redirections OAuth.

---

## Niveaux de variables d'environnement

### Niveau 0 — Coque de l'app seulement

| Variable | Requis ? |
| --- | --- |
| `DATABASE_URL` | **Oui** pour les fonctions liées à la DB (expand, share, cache connections) |
| `NEXT_PUBLIC_APP_URL` | Recommandé |

**Fonctionne sans clés IA :** Pages statiques/marketing, coque UI du canvas, canvas localStorage (client seulement).

**INFERENCE:** Le fetch de verset peut marcher via le fallback alquran.cloud si le corpus n'est pas chargé — mais expand/share/graphe touchent Postgres et échouent si la DB est down.

### Niveau 1 — Produit principal (défaut contributeur)

| Variable | Active |
| --- | --- |
| `ANTHROPIC_API_KEY` (+ `AI_PROVIDER=claude`) | Expand IA, raisons de connexion |
| `GEMINI_API_KEY` | Embed de requête pour recherche sémantique ; requis pour `embed-corpus` |
| `verses` chargé + migrations | Corpus local rapide, parité validation avec CI |

### Niveau 2 — Auth + social + workspaces

| Variable | Active |
| --- | --- |
| `NEXT_PUBLIC_QF_CLIENT_ID` | ID client OAuth navigateur |
| `QF_CLIENT_SECRET` | Échange de token serveur |
| `QF_AUTH_BASE` / `NEXT_PUBLIC_QF_AUTH_BASE` | Prelive : `https://prelive-oauth2.quran.foundation` |
| Redirect URI enregistré | `http://localhost:3000/callback` avec Quran Foundation |

**FACT:** `.env.example` documente le QF **prelive** pour le dev local ; l'OAuth production a un enablement de fonctionnalités plus strict.

### Niveau 2b — Bypass auth dev (sans OAuth QF pour l'instant)

Quand le redirect QF n'est pas enregistré :

```bash
DEV_AUTH_TOKEN=<long-random-secret>
DEV_AUTH_QF_ID=dev-admin
DEV_AUTH_USERNAME=devadmin   # optional
ADMIN_QF_IDS=dev-admin       # if testing /admin
```

Dans la console du navigateur (dev seulement) :

```javascript
await window.__devLogin('<DEV_AUTH_TOKEN>')
```

**FACT:** Désactivé en dur quand `NODE_ENV=production` (`lib/auth/social-auth.ts`, `components/providers.tsx`).

### Niveau 3 — Accélérateurs optionnels

| Variable | Rôle |
| --- | --- |
| `REDIS_URL` | Cache embed de requête, rate limiter, cache token — l'app marche sans |
| `GEMINI_API1`…`GEMINI_API5` | Boucle backfill admin seulement — pas le trafic live |
| `ADMIN_QF_IDS` | Accès console `/admin` |

---

## Matrice fonctionnalité × exigences

| Fonctionnalité | Postgres | Corpus seed | Embeddings | Clé IA | Auth |
| --- | --- | --- | --- | --- | --- |
| Home / marketing | No | No | No | No | No |
| Canvas + localStorage | No* | No* | No | No | No |
| Recherche par ref `2:255` | No* | Prefer yes | No | No | No |
| Recherche mot-clé | No | Prefer yes | No | No | No |
| Liés par le sens | **Yes** | **Yes** | **Yes** | Gemini embed | No |
| Expand (Theme/Root/Contrast) | **Yes** | **Yes** | Root/embed par kind | **Yes** | No |
| URL partage canvas | **Yes** | No | No | No | No |
| Sync bookmarks | **Yes** | No | No | No | QF OAuth ou dev auth |
| Workspaces sauvegardés | **Yes** | No | No | No | Auth |
| Panneau admin | **Yes** | Varies | Varies | Varies | `ADMIN_QF_IDS` |

\* Canvas et recherche par ref peuvent partiellement marcher via état client + fallback alquran.cloud live, mais ce n'est pas représentatif de la production ou CI.

---

## Vérifier votre setup (checklist smoke)

Après les étapes 1–8 :

| # | Vérification | Comment | Critère de succès |
| --- | --- | --- | --- |
| 1 | Santé | `curl -s http://localhost:3000/api/health` | `{"status":"ok"}` |
| 2 | Home | Ouvrir `/` | Page charge, pas de 500 |
| 3 | Canvas | Ouvrir `/canvas` | État vide ou graphe restauré depuis localStorage |
| 4 | API verset | `curl -s http://localhost:3000/api/verse/2/255` | JSON avec `ref`, `arabicText`, `translation` |
| 5 | Recherche | ⌘K → chercher `2:255` ou `mercy` | Résultats apparaissent |
| 6 | Expand | Ajouter `2:255` → expand Theme | Nouveaux nœuds + arêtes (nécessite niveau 1) |
| 7 | Sémantique | Recherche avec phrase concept ; vérifier « related » | Nécessite embeddings + `GEMINI_API_KEY` |
| 8 | Partage | Toolbar Share (avec nœuds sur le canvas) | URL copiée ; ouvre dans un nouvel onglet (nécessite Postgres) |

**FACT:** `/api/health` retourne OK statique — il **ne vérifie pas** la connectivité Postgres.

**INFERENCE:** Health check OK + expand qui échoue signifie souvent **DB down**, **seed manquant**, ou **clé IA manquante** — pas une install Next.js cassée.

---

## Redis (optionnel)

**FACT:** `.env.example` : laisser `REDIS_URL` non défini → replis in-process / Postgres (`lib/infra/redis.ts`).

**FACT:** `docker compose up` démarre Redis et configure `REDIS_URL` automatiquement.

**UTILE PLUS TARD :** Redis réduit les appels Gemini embed répétés pour les requêtes populaires (cache 7 jours dans `semantic-search.ts`).

---

## Lancer les tests en local (pas la profondeur phase 10, mais utile avant PR)

| Commande | Besoin | Notes |
| --- | --- | --- |
| `bun run test` / `test:ci` | Rien externe | Tests unitaires ; mock DB |
| `bun run test:integration` | **Docker en marche** | Testcontainers lance Postgres pgvector |
| `bun run test:e2e` | Docker Postgres + migrate + `DEV_AUTH_*` dans `.env.local` | Playwright sur port **3100** |
| `bun run lint` / `typecheck` / `format:check` | Rien | Parité CI |

**FACT:** `.husky/pre-push` lance les tests d'intégration — **`docker ps` doit réussir avant `git push`** (AGENTS.md).

**FACT:** E2E CI lance `db:migrate` + `db:migrate:concurrent-indexes` mais **ne lance pas** le seed corpus complet — les tests e2e utilisent dev auth + chemins live/fallback adaptés à l'env CI.

---

## Échecs courants (diagnostic d'abord)

| Symptôme | Cause probable | Correction |
| --- | --- | --- |
| `connection refused` sur expand/share | Postgres ne tourne pas ou mauvais `DATABASE_URL` | Démarrer conteneur pgvector ; vérifier URL |
| Expand retourne toast d'erreur immédiat | `ANTHROPIC_API_KEY` manquant / mauvais provider | Définir clés niveau 1 ; redémarrer dev server |
| Expand réussit mais toujours legacy/lent | Pas de morphologie/embeddings chargés | Lancer scripts seed phase 9 |
| « Related by meaning » n'apparaît jamais | Pas de `verse_embeddings` ou pas de `GEMINI_API_KEY` | `bun run embed` + clé |
| Erreur redirect OAuth | `NEXT_PUBLIC_APP_URL` ne correspond pas ou callback non enregistré | Utiliser `http://localhost:3000` ; enregistrer callback avec QF |
| Sign-in marche mais pas admin | Utilisateur pas dans `ADMIN_QF_IDS` | Ajouter votre `DEV_AUTH_QF_ID` ou QF `sub` |
| Hook `git push` échoue mystérieusement | Docker pas lancé pour Testcontainers | Démarrer Docker Desktop |
| Env changé, comportement inchangé | Dev server pas redémarré | Arrêter et `bun run dev` à nouveau |
| Recherche sémantique marche une fois puis rapide | Cache Redis — attendu | Pas une erreur |

**À COMPRENDRE MAINTENANT :** N'inventez pas de contournements (skip hooks, mock DB prod) avant de vérifier le tableau ci-dessus — les échecs de ce dépôt sont souvent **service manquant ou seed manquant**, pas des bugs Next.js mystérieux.

---

## Starter `.env.local` minimal (copier et remplir)

```bash
# ─── Required for DB-backed dev ───
DATABASE_URL=postgresql://openh:devpassword@localhost:5432/open_hikmah
NEXT_PUBLIC_APP_URL=http://localhost:3000

# ─── AI expand ───
ANTHROPIC_API_KEY=
AI_PROVIDER=claude

# ─── Semantic search + embed-corpus ───
GEMINI_API_KEY=

# ─── Dev auth (optional — skip QF OAuth early) ───
# DEV_AUTH_TOKEN=
# DEV_AUTH_QF_ID=dev-admin
# ADMIN_QF_IDS=dev-admin

# ─── QF OAuth (optional until testing bookmarks/sync) ───
# NEXT_PUBLIC_QF_CLIENT_ID=
# QF_CLIENT_SECRET=
# QF_AUTH_BASE=https://prelive-oauth2.quran.foundation
# NEXT_PUBLIC_QF_AUTH_BASE=https://prelive-oauth2.quran.foundation
```

---

## Séquence complète première fois (copier-coller)

Suppose Docker installé, Bun installé, `.env.local` rempli avec clés IA :

```bash
bun install
cp .env.example .env.local
# edit .env.local

docker run -d --name openhikmah-db \
  -e POSTGRES_DB=open_hikmah \
  -e POSTGRES_USER=openh \
  -e POSTGRES_PASSWORD=devpassword \
  -p 5432:5432 \
  pgvector/pgvector:pg16

bun run db:migrate
bun run db:migrate:concurrent-indexes
bun scripts/seed-quran.mjs
bun scripts/seed-morphology.mjs
bun run embed

bun run dev
# open http://localhost:3000/canvas
```

---

## À COMPRENDRE MAINTENANT

1. **Bun** est le gestionnaire de paquets et lanceur de scripts du projet.
2. **Postgres + pgvector** est requis pour expand, share, cache graphe et classement sémantique — pas optionnel pour le dev produit principal.
3. **Ordre de seed :** migrate → quran → morphology → embed (phase 9).
4. **`ANTHROPIC_API_KEY`** (ou provider Gemini) pour expand ; **`GEMINI_API_KEY`** en plus pour sémantique/embed.
5. **Redémarrer le dev server** après changements `.env.local`, surtout `NEXT_PUBLIC_*`.
6. **Docker** requis pour `test:integration` et hook pre-push — pas forcément pour `bun run dev` seul.

---

## UTILE PLUS TARD

- `docker compose up` — stack complète type production (app + db + redis).
- `bun run prewarm` — préchauffage offline du graphe contre l'app en marche.
- `scripts/seed-translations.mjs` — lignes d'éditions non anglaises.
- Playwright : `bun run test:e2e` sur port 3100 avec `DEV_AUTH_*` défini.

---

## IGNORER POUR L'INSTANT

- Déploiement production (`Dockerfile`, `scripts/deploy.sh`) — sauf contributions infra.
- Pool `GEMINI_API1`–`5` — boucle backfill admin seulement.
- Tuning paramètres Postgres/HNSW — les défauts marchent pour le dev local.

---

## Questions de contrôle — Phase 10

1. Quels trois scripts de seed lancer après migrate pour expand ancré complet + recherche sémantique ?
2. Pourquoi `GEMINI_API_KEY` est requis même quand `AI_PROVIDER=claude` ?
3. Quelle URL et quelles variables d'env QF OAuth nécessite pour la connexion locale ?
4. Comment bypasser OAuth en dev, et pourquoi ça ne marche pas en build production ?
5. Quelle commande prouve que le processus Next.js tourne — et que ne prouve-t-elle pas ?

---

**Suite :** [Phase 11 — Corrélation navigateur ↔ code](./phase-11-browser-code-correlation.md) — faire un flux runtime dans le navigateur et relier l'onglet Network → API → bibliothèque → base de données.

Dites **« continuer vers la Phase 11 »** quand vous avez suivi cette checklist (ou si vous avez un échec précis à déboguer), ou demandez le détail d'une étape.

> **Version complète (français B2+) :** [Phase 10](../onboarding-fr/phase-10-run-locally.md)

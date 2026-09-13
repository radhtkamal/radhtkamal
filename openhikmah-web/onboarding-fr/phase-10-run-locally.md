# Phase 10 — Lancer l'application localement

> **Prérequis :** Phases [1](./phase-1-what-is-openhikmah.md)–[9](./phase-9-database-and-semantic-search.md)  
> **Étiquettes de preuve :** **FACT** (fait établi) · **INFERENCE** (inférence) · **UNKNOWN** (inconnu)

La phase 9 a expliqué **ce que Postgres stocke**. La phase 10 est une **configuration pratique** — comment exécuter OpenHikmah sur votre machine, quelles variables d'environnement débloquent quelles fonctionnalités, et comment distinguer le succès d'une mauvaise configuration.

**Cette phase documente le workflow.** Exécutez les commandes vous-même quand vous êtes prêt ; rien ici ne modifie le code applicatif.

---

## Ce que vous construisez localement

```mermaid
flowchart LR
  subgraph machine ["Votre machine"]
    Bun[bun run dev :3000]
    Browser[Navigateur]
    LS[(localStorage canvas)]
  end

  subgraph services ["Services locaux"]
    PG[(Postgres + pgvector)]
    Redis[(Redis — optionnel)]
  end

  subgraph external ["API externes"]
    Claude[Anthropic Claude]
    Gemini[Gemini embed + LLM optionnel]
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

**À COMPRENDRE MAINTENANT :** OpenHikmah n'est **pas** un frontend statique. Recherche, expand, partage, auth et le graphe IA ont tous besoin **des routes serveur + Postgres** pour un comportement complet.

---

## Pourquoi Bun ?

**FACT:** `package.json` déclare `"packageManager": "bun@1"`. CI et le Dockerfile utilisent `oven-sh/setup-bun` / `oven/bun:1-alpine`.

| Raison | Détail |
| --- | --- |
| Chaîne d'outils officielle | `bun install`, `bun run dev`, scripts via `bun scripts/...` |
| Vitesse | Installations rapides — important sur un grand arbre de dépendances Next.js |
| Lockfile | `bun.lock` — CI utilise `bun install --frozen-lockfile` |

**FACT:** Vous n'avez pas besoin de Node/npm pour le travail quotidien si Bun est installé. Playwright e2e utilise toujours `bunx playwright install`.

**UNKNOWN:** Si les mainteneurs testent sur une version patch spécifique de Bun au-delà de `@1` — CI utilise `latest`.

---

## Guide pas à pas (étapes ordonnées)

Chaque étape : **quoi**, **changement d'état**, **succès**, **échec courant**.

### Étape 1 — Cloner et installer les dépendances

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

OpenHikmah requiert **PostgreSQL 16 + pgvector** (`docker-compose.yml`, migration `0008`).

**Option A — Docker Compose (app + db + redis) :**

```bash
# Définir DB_PASSWORD dans .env ou le shell, puis :
docker compose up -d db
# Optionnel : docker compose up -d   # stack complète
```

**Option B — Postgres uniquement (correspond à l'URL de `.env.example`) :**

```bash
docker run -d --name openhikmah-db \
  -e POSTGRES_DB=open_hikmah \
  -e POSTGRES_USER=openh \
  -e POSTGRES_PASSWORD=devpassword \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

Définir dans `.env.local` :

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

```bash
bun run db:migrate
bun run db:migrate:concurrent-indexes   # index locale connections — une fois en local
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

```bash
bun run scripts/seed-quran.mjs
# ou : node scripts/seed-quran.mjs  (scripts en .mjs ; package.json a des wrappers bun pour certains)
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

```bash
bun run scripts/seed-morphology.mjs   # word_morphology depuis data/morphology/*.jsonl
bun run embed                         # alias : scripts/embed-corpus.mjs — nécessite GEMINI_API_KEY
```

| Script | Débloque |
| --- | --- |
| `seed-morphology.mjs` | Découverte ancrée **Racine** (chemin racine de `discoverCandidates`) |
| `embed-corpus.mjs` | Découverte **Thématique/contrast** + recherche sémantique + « liés par le sens » |

| | |
| --- | --- |
| **Quoi** | Tables d'ancrage niveau 1 (phase 9) |
| **Changement d'état** | `word_morphology`, `verse_embeddings` remplies |
| **Succès** | Logs morphologie par fichier ; embed log `verse_embeddings table now holds N rows` |
| **Échec** | Embed sans `GEMINI_API_KEY` → le script sort ; limite 429 → relancer plus tard (idempotent) |

**FACT:** `bun run embed` est défini dans `package.json` comme `bun scripts/embed-corpus.mjs`.

---

### Étape 7 — Définir les clés IA (minimum pour expand)

Dans `.env.local`, au minimum pour **expand de connexion IA** :

```bash
ANTHROPIC_API_KEY=sk-ant-...
AI_PROVIDER=claude
```

**Alternative dev économique :**

```bash
AI_PROVIDER=gemini
GEMINI_API_KEY=...
```

**FACT:** `AGENTS.md` / `CONTRIBUTING.md` : *« Au minimum vous avez besoin de `ANTHROPIC_API_KEY` pour tester les connexions IA. »* Gemini fonctionne quand `AI_PROVIDER=gemini`.

**Pour la recherche sémantique / embed corpus :**

```bash
GEMINI_API_KEY=...   # requis même quand AI_PROVIDER=claude
```

Les embeddings sont **toujours Gemini** (`lib/ai/ai.ts`).

---

### Étape 8 — Démarrer le serveur de dev

```bash
bun run dev
```

| | |
| --- | --- |
| **Quoi** | `next dev` — serveur de dev Next.js 16 App Router |
| **Changement d'état** | Processus à l'écoute sur le port 3000 (par défaut) |
| **Succès** | Le terminal affiche ready ; [http://localhost:3000](http://localhost:3000) se charge |
| **Échec** | Port 3000 occupé → `bun run dev -- -p 3001` et mettre à jour `NEXT_PUBLIC_APP_URL` ; dépendances manquantes → relancer `bun install` |

**FACT:** Définir `NEXT_PUBLIC_APP_URL=http://localhost:3000` dans `.env.local` pour la cohérence des redirections OAuth.

---

## Niveaux de variables d'environnement

### Niveau 0 — Coquille applicative uniquement

| Variable | Requise ? |
| --- | --- |
| `DATABASE_URL` | **Oui** pour les fonctionnalités DB (expand, partage, cache connections) |
| `NEXT_PUBLIC_APP_URL` | Recommandée |

**Fonctionne sans clés IA :** Pages statiques/marketing, coquille UI canvas, canvas localStorage (client uniquement).

**INFERENCE:** La récupération de versets peut fonctionner via le repli alquran.cloud si le corpus n'est pas chargé — mais expand/partage/routes graphe touchent Postgres et échouent si la DB est down.

### Niveau 1 — Produit principal (défaut contributeur)

| Variable | Active |
| --- | --- |
| `ANTHROPIC_API_KEY` (+ `AI_PROVIDER=claude`) | Expand IA, raisons de connexion |
| `GEMINI_API_KEY` | Embedding de requête pour recherche sémantique ; requis pour `embed-corpus` |
| `verses` chargé + migrations | Corpus local rapide, parité validation avec CI |

### Niveau 2 — Auth + social + workspaces

| Variable | Active |
| --- | --- |
| `NEXT_PUBLIC_QF_CLIENT_ID` | Id client OAuth navigateur |
| `QF_CLIENT_SECRET` | Échange de token serveur |
| `QF_AUTH_BASE` / `NEXT_PUBLIC_QF_AUTH_BASE` | Prelive : `https://prelive-oauth2.quran.foundation` |
| Redirect URI enregistrée | `http://localhost:3000/callback` avec Quran Foundation |

**FACT:** `.env.example` documente QF **prelive** pour le dev local ; OAuth production a un enablement de fonctionnalités plus strict.

### Niveau 2b — Contournement auth dev (pas encore QF OAuth)

Quand la redirect QF n'est pas enregistrée :

```bash
DEV_AUTH_TOKEN=<long-random-secret>
DEV_AUTH_QF_ID=dev-admin
DEV_AUTH_USERNAME=devadmin   # optionnel
ADMIN_QF_IDS=dev-admin       # si test de /admin
```

Dans la console navigateur (dev uniquement) :

```javascript
await window.__devLogin('<DEV_AUTH_TOKEN>')
```

**FACT:** Désactivé en dur quand `NODE_ENV=production` (`lib/auth/social-auth.ts`, `components/providers.tsx`).

### Niveau 3 — Accélérateurs optionnels

| Variable | Rôle |
| --- | --- |
| `REDIS_URL` | Cache embedding requête, rate limiter, cache token — l'app fonctionne sans |
| `GEMINI_API1`…`GEMINI_API5` | Boucle backfill admin uniquement — pas le trafic live |
| `ADMIN_QF_IDS` | Accès console `/admin` |

---

## Matrice fonctionnalité × prérequis

| Fonctionnalité | Postgres | Seed corpus | Embeddings | Clé IA | Auth |
| --- | --- | --- | --- | --- | --- |
| Accueil / marketing | Non | Non | Non | Non | Non |
| Canvas + localStorage | Non* | Non* | Non | Non | Non |
| Recherche par ref `2:255` | Non* | Préférable oui | Non | Non | Non |
| Recherche par mot-clé | Non | Préférable oui | Non | Non | Non |
| Liés par le sens | **Oui** | **Oui** | **Oui** | Gemini embed | Non |
| Expand (Theme/Root/Contrast) | **Oui** | **Oui** | Root/embed selon kind | **Oui** | Non |
| URL partage canvas | **Oui** | Non | Non | Non | Non |
| Sync signets | **Oui** | Non | Non | Non | QF OAuth ou dev auth |
| Workspaces sauvegardés | **Oui** | Non | Non | Non | Auth |
| Panneau admin | **Oui** | Variable | Variable | Variable | `ADMIN_QF_IDS` |

\* Canvas et recherche par ref peuvent partiellement fonctionner via état client + repli alquran.cloud live, mais ce n'est pas représentatif de la production ou CI.

---

## Vérifier votre configuration (checklist smoke)

À exécuter après les étapes 1–8 :

| # | Vérification | Comment | Critères de succès |
| --- | --- | --- | --- |
| 1 | Santé | `curl -s http://localhost:3000/api/health` | `{"status":"ok"}` |
| 2 | Accueil | Ouvrir `/` | Page se charge, pas de 500 |
| 3 | Canvas | Ouvrir `/canvas` | État vide ou graphe localStorage restauré |
| 4 | API verset | `curl -s http://localhost:3000/api/verse/2/255` | JSON avec `ref`, `arabicText`, `translation` |
| 5 | Recherche | ⌘K → rechercher `2:255` ou `mercy` | Résultats apparaissent |
| 6 | Expand | Ajouter `2:255` → expand Theme | Nouveaux nœuds + arêtes (nécessite niveau 1) |
| 7 | Sémantique | Recherche avec phrase conceptuelle ; vérifier « related » | Nécessite embeddings + `GEMINI_API_KEY` |
| 8 | Partage | Toolbar Share (avec nœuds sur le canvas) | URL copiée ; s'ouvre dans un nouvel onglet (nécessite Postgres) |

**FACT:** `/api/health` renvoie OK statique — il **ne** vérifie **pas** la connectivité Postgres.

**INFERENCE:** Health check OK + expand en échec signifie généralement **DB down**, **seed manquant** ou **clé IA manquante** — pas une installation Next.js cassée.

---

## Redis (optionnel)

**FACT:** `.env.example` : laisser `REDIS_URL` non défini → replis in-process / Postgres (`lib/infra/redis.ts`).

**FACT:** `docker compose up` démarre Redis et configure `REDIS_URL` automatiquement.

**UTILE PLUS TARD :** Redis réduit les appels Gemini embed répétés pour les requêtes de recherche populaires (cache 7 jours dans `semantic-search.ts`).

---

## Exécuter les tests localement (pas la profondeur phase 10, mais nécessaire avant PR)

| Commande | Nécessite | Notes |
| --- | --- | --- |
| `bun run test` / `test:ci` | Rien d'externe | Tests unitaires ; mock DB |
| `bun run test:integration` | **Docker en cours** | Testcontainers lance Postgres pgvector |
| `bun run test:e2e` | Docker Postgres + migrate + `DEV_AUTH_*` dans `.env.local` | Playwright sur le port **3100** |
| `bun run lint` / `typecheck` / `format:check` | Rien | Parité CI |

**FACT:** `.husky/pre-push` exécute les tests d'intégration — **`docker ps` doit réussir avant `git push`** (AGENTS.md).

**FACT:** E2E CI exécute `db:migrate` + `db:migrate:concurrent-indexes` mais **n'exécute pas** le seed corpus complet — les tests e2e utilisent dev auth + chemins live/repli adaptés à l'env CI.

---

## Échecs courants (diagnostic d'abord)

| Symptôme | Cause probable | Correction |
| --- | --- | --- |
| `connection refused` sur expand/partage | Postgres non démarré ou mauvais `DATABASE_URL` | Démarrer le conteneur pgvector ; vérifier l'URL |
| Expand renvoie toast d'erreur immédiatement | `ANTHROPIC_API_KEY` manquante / mauvais provider | Définir clés niveau 1 ; redémarrer le serveur de dev |
| Expand réussit mais toujours legacy/lent | Pas de morphologie/embeddings chargés | Exécuter scripts seed phase 9 |
| « Liés par le sens » n'apparaît jamais | Pas de `verse_embeddings` ou pas de `GEMINI_API_KEY` | `bun run embed` + clé |
| Erreur redirect OAuth | `NEXT_PUBLIC_APP_URL` incohérent ou callback non enregistré | Utiliser `http://localhost:3000` ; enregistrer callback avec QF |
| Connexion OK mais pas admin | Utilisateur pas dans `ADMIN_QF_IDS` | Ajouter votre `DEV_AUTH_QF_ID` ou `sub` QF |
| Hook `git push` échoue mystérieusement | Docker non démarré pour Testcontainers | Démarrer Docker Desktop |
| Env modifiée, comportement inchangé | Serveur de dev non redémarré | Tuer et `bun run dev` à nouveau |
| Recherche sémantique rapide après la première fois | Cache Redis — attendu | Pas une erreur |

**À COMPRENDRE MAINTENANT :** N'inventez pas de contournements (skip hooks, mock DB prod) avant de vérifier le tableau ci-dessus — les échecs de ce dépôt sont généralement **service manquant ou seed manquant**, pas des bugs Next.js mystérieux.

---

## Starter minimal `.env.local` (copier et remplir)

```bash
# ─── Requis pour dev DB-backed ───
DATABASE_URL=postgresql://openh:devpassword@localhost:5432/open_hikmah
NEXT_PUBLIC_APP_URL=http://localhost:3000

# ─── Expand IA ───
ANTHROPIC_API_KEY=
AI_PROVIDER=claude

# ─── Recherche sémantique + embed-corpus ───
GEMINI_API_KEY=

# ─── Auth dev (optionnel — skip QF OAuth au début) ───
# DEV_AUTH_TOKEN=
# DEV_AUTH_QF_ID=dev-admin
# ADMIN_QF_IDS=dev-admin

# ─── QF OAuth (optionnel jusqu'aux tests signets/sync) ───
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
# éditer .env.local

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
# ouvrir http://localhost:3000/canvas
```

---

## À COMPRENDRE MAINTENANT

1. **Bun** est le gestionnaire de paquets et l'exécuteur de scripts du projet.
2. **Postgres + pgvector** est requis pour expand, partage, cache graphe et classement sémantique — pas optionnel pour le dev produit principal.
3. **Ordre de seed :** migrate → quran → morphology → embed (phase 9).
4. **`ANTHROPIC_API_KEY`** (ou provider Gemini) pour expand ; **`GEMINI_API_KEY`** en plus pour chemins sémantique/embed.
5. **Redémarrer le serveur de dev** après changements `.env.local`, surtout `NEXT_PUBLIC_*`.
6. **Docker** requis pour `test:integration` et le hook pre-push — pas nécessairement pour `bun run dev` lui-même.

---

## UTILE PLUS TARD

- `docker compose up` — stack complète type production (app + db + redis).
- `bun run prewarm` — préchauffage offline du graphe contre l'app en cours.
- `scripts/seed-translations.mjs` — lignes édition non anglaise.
- Playwright : `bun run test:e2e` sur le port 3100 avec `DEV_AUTH_*` défini.

---

## IGNORER POUR L'INSTANT

- Déploiement production (`Dockerfile`, `scripts/deploy.sh`) — sauf contributions infra.
- Pool `GEMINI_API1`–`5` — boucle backfill admin uniquement.
- Paramètres tuning Postgres/HNSW — les défauts suffisent pour le dev local.

---

## Questions de contrôle — Phase 10

1. Quels trois scripts de seed devez-vous exécuter après migrate pour expand ancré complet + recherche sémantique ?
2. Pourquoi `GEMINI_API_KEY` est-elle requise même quand `AI_PROVIDER=claude` ?
3. Quelle URL et quelles variables d'env QF OAuth nécessite-t-il pour la connexion locale ?
4. Comment contourner OAuth en dev, et pourquoi cela ne fonctionne-t-il pas en builds production ?
5. Quelle commande prouve que le processus Next.js est up — et que ne prouve-t-elle *pas* ?

---

**Suite :** [Phase 11 — Corrélation navigateur ↔ code](./phase-11-browser-code-correlation.md) — effectuer un flux runtime dans le navigateur et mapper l'onglet Network → API → bibliothèque → base de données.

Dites **« continuer vers la Phase 11 »** quand vous avez parcouru cette checklist (ou rencontré un échec spécifique à déboguer), ou demandez un détail sur n'importe quelle étape.

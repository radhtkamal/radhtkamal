# Intégration uMap — Phase 1 : Qu'est-ce que uMap ?

> **Statut :** Phase 1 sur 13 · Investigation en lecture seule · Preuves issues du dépôt consulté  
> **Public :** Ingénieur web expérimenté (React/TS/Node) qui apprend uMap pour contribuer de manière significative à l'OSS

---

## Avant d'examiner les fichiers

Voici le modèle mental à garder en tête en parcourant le dépôt.

**uMap est une application web pour créer et publier des cartes thématiques.** Un utilisateur part d'un fond de carte basé sur OpenStreetMap, ajoute son propre contenu géographique (marqueurs, lignes, polygones, jeux de données importés), organise ce contenu en calques, le stylise, contrôle qui peut le voir ou le modifier, puis partage ou intègre le résultat sur un site web.

Ce n'est **pas** un outil SIG de bureau à usage général, **pas** un éditeur OpenStreetMap (comme iD ou JOSM), et **pas** simplement un visualiseur de cartes. Le produit occupe un créneau plus étroit : **la narration cartographique web rapide et la visualisation légère de données**, visant surtout les personnes qui veulent OSM comme toile de fond mais ont besoin d'une superposition personnalisée qu'elles peuvent publier en quelques minutes.

Pensez-y ainsi si vous venez de React/Firebase :

| Concept familier | Équivalent uMap |
|---|---|
| Un projet SPA hébergé sur Firebase | Une **carte** — le document de premier niveau que les utilisateurs créent |
| Collections / sous-collections Firestore | **Calques de données** — entités géographiques regroupées dans une carte |
| Documents avec champs JSON | **Entités** — points, lignes, polygones avec propriétés (nom, description, champs personnalisés) |
| Règles de sécurité (lecture/écriture) | **Statut de partage** et **statut d'édition** sur les cartes et les calques |
| Intégration d'un widget sur un autre site | **Intégration iframe** avec paramètres d'URL contrôlant les commandes |

Cette comparaison est volontairement approximative — uMap est un Django rendu côté serveur plus un client Leaflet, pas une app Firebase — mais elle capture la *forme du produit* : **un artefact créé, des groupes de données imbriqués, des permissions, publier/intégrer**.

---

## Ce qu'est uMap

**FAITE :** Le README du projet décrit uMap ainsi : *« lets you create maps with OpenStreetMap layers within minutes and embed them in your site. »* Il est construit sur **Django** (serveur) et **Leaflet** (bibliothèque cartographique côté client).

**FAITE :** Les mots-clés de `pyproject.toml` incluent `django`, `leaflet`, `geodjango`, `openstreetmap`, `map`.

**FAITE :** La vue d'ensemble développeur (`docs/dev/overview.md`) indique que uMap est **un serveur et un client**. La plupart des données géographiques sont stockées sous forme de **fichiers GeoJSON** sur le serveur ; les utilisateurs, les permissions et les métadonnées associées vivent dans une **base de données** (PostgreSQL avec PostGIS pour certaines fonctionnalités géo). La plupart des calculs cartographiques se font sur le **frontend** avec Leaflet.

**INFÉRENCE :** uMap est conçu pour être **auto-hébergeable** (docs d'installation, Docker, Helm, etc.) tout en fonctionnant aussi comme **service public hébergé** — l'instance la plus connue est [umap.openstreetmap.fr](https://umap.openstreetmap.fr/), référencée dans les tutoriels utilisateur.

---

## Le problème qu'il résout

OpenStreetMap offre au monde une toile de fond partagée et éditable. Mais beaucoup de personnes — associations, journalistes, collectivités, organisateurs d'événements, militants — ont besoin de quelque chose de plus spécifique :

- « Afficher *nos* lieux de festival par-dessus OSM »
- « Tracer un itinéraire vélo et le partager »
- « Importer les emplacements des bibliothèques et laisser les visiteurs basculer les calques »
- « Intégrer une carte interactive sur notre site WordPress »

**FAITE :** La ligne de motivation du README est explicite : *« Because we think that the more OSM will be used, the more OSM will be improved. »*

uMap abaisse la barrière entre **avoir des géodonnées** et **publier une carte interactive**. Les utilisateurs n'ont pas besoin de logiciel SIG, de serveurs de tuiles ni de code Leaflet personnalisé. Ils obtiennent outils de dessin, assistants d'import, style, permissions et codes d'intégration dans un seul produit.

---

## Qui l'utilise

**FAITE (d'après la documentation et les tutoriels utilisateur) :** Les segments d'utilisateurs documentés incluent :

- **Grand public et associations** — utilisant l'instance communautaire OSM (`umap.openstreetmap.fr`)
- **Agents du secteur public (France)** — une instance dédiée (`umap.incubateur.anct.gouv.fr`) avec authentification ProConnect
- **Toute personne intégrant des cartes** — iframe HTML, WordPress, etc. (tutoriel 7)
- **Auteurs de cartes travaillant avec des données OpenStreetMap** — requêtes Overpass, imports GeoDataMine, limites administratives (tutoriels 6, 11)

**INFÉRENCE :** Les utilisateurs typiques sont des **non-programmeurs** qui ont besoin de cartes publiables, plus des **utilisateurs techniques** qui veulent une cartographie thématique rapide sans construire une application sur mesure. En tant que contributeur, vous êtes du côté ingénierie d'un produit principalement façonné par les besoins des cartographes, journalistes et associations.

**INCONNU (à partir de ce dépôt seul) :** Statistiques d'usage exactes, distribution géographique ou répartition commercial vs. communautaire. Le projet est financé par la communauté (badges Liberapay, OpenCollective, GitHub Sponsors dans le README).

---

## Parcours utilisateur les plus importants

Ce sont les flux à garder en tête. Nous les retracerons dans le code en Phase 5.

### 1. Consulter / consommer une carte (lecteur)

**FAITE :** Un lecteur ouvre une URL partagée, se déplace et zoome, active/désactive les calques, clique sur les entités pour voir les popups (texte, images, liens), et peut ouvrir un panneau latéral « À propos » ou « Parcourir les données ».

C'est l'expérience **en lecture seule** — aucun compte requis pour les cartes publiques.

### 2. Créer et modifier une carte (auteur)

**FAITE :** Les auteurs entrent en **mode édition** (`Ctrl+E` selon la FAQ). En mode édition, ils peuvent :

- Nommer la carte et rédiger une description/légende
- Dessiner des **marqueurs**, des **lignes**, des **polygones** et des itinéraires
- Configurer les couches de tuiles/fond de carte et les paramètres de la carte
- **Enregistrer** (`Ctrl+S`)
- **Annuler** jusqu'au dernier enregistrement (`Ctrl+Z`)

**FAITE :** Les cartes peuvent être créées **sans compte**. Les cartes anonymes reçoivent un **lien d'édition secret** ; seules les personnes disposant de ce lien peuvent les modifier (tutoriel 2, français).

### 3. Organiser le contenu en calques

**FAITE :** Les entités vivent dans des **calques de données** (calques). Les calques peuvent être affichés/masqués, réordonnés, décrits dans la légende, et ciblés lors du dessin ou de l'import. Une carte peut présenter différents sous-ensembles de calques dans différentes « vues » d'intégration (tutoriel 6).

**À COMPRENDRE MAINTENANT :** **Carte → Calque(s) de données → Entité(s)** est la hiérarchie de contenu centrale.

### 4. Importer des données externes

**FAITE :** uMap propose des assistants d'import pour des éléments comme les limites administratives françaises, les jeux de données POI GeoDataMine, l'upload de fichiers (GeoJSON, etc.) et les requêtes **Overpass** contre OpenStreetMap (tutoriels 6, 11).

Une distinction importante du tutoriel 11 :

- **Copié dans le calque** — les données sont stockées dans uMap (dans le fichier GeoJSON du calque)
- **Données distantes** — les données ne sont **pas stockées sur le serveur uMap** ; le client les récupère en direct (ex. API Overpass, URL GeoJSON distante)

### 5. Publier, partager, intégrer

**FAITE :** Les auteurs contrôlent le **statut d'accès** (brouillon, public, lien uniquement, éditeurs uniquement) et le **statut d'édition** (propriétaire, collaborateurs, tout le monde, lien secret). Ils peuvent exporter/partager via une intégration iframe avec de nombreux paramètres d'URL (échelle, contrôle des calques, mode édition, etc.) — tutoriel 7.

**FAITE :** Les lecteurs non autorisés obtiennent **403 Forbidden** (FAQ).

### 6. Catalogue basé sur un compte (optionnel)

**FAITE :** Les utilisateurs peuvent créer des comptes pour maintenir un catalogue personnel de cartes au lieu de s'appuyer sur des liens d'édition secrets (tutoriel 3, référencé depuis le tutoriel 2).

**UTILE PLUS TARD :** Équipes, collaborateurs, modèles — nous les verrons dans le modèle de données lorsque nous aborderons l'architecture.

---

## Quel type de données les utilisateurs manipulent

### La carte (document de premier niveau)

**FAITE :** Dans `umap/models.py`, une `Map` possède : nom, slug, point central, zoom, propriétaire, éditeurs, équipe, statuts de partage/édition, tags, licence, et un champ JSON `settings` pour la configuration au niveau de la carte (description, couches de tuiles, options d'interface, etc.).

Pensez : **la toile + les métadonnées + l'enveloppe de permissions**.

### Calques de données

**FAITE :** Un `DataLayer` appartient à une `Map`, possède une clé primaire UUID, un nom, une description, un rang (ordre), un JSON de paramètres, des permissions, et un **champ fichier `geojson`** — les entités du calque sont principalement persistées sous forme de fichier GeoJSON sur disque (ou stockage objet).

Pensez : **un groupe nommé d'entités**, comme un calque dans Photoshop ou une collection dans Firestore.

### Entités (dans le GeoJSON)

**FAITE :** Les entités sont des types de géométrie GeoJSON standard — **Point** (marqueurs), **LineString** (lignes/itinéraires), **Polygon** (zones) — avec des propriétés : nom, description, colonnes personnalisées, indices de style.

**FAITE :** La FAQ documente les **règles de style conditionnel** sur les propriétés des entités (`population>10000` → couleur rouge), les **variables de modèle** (`{lat}`, `{name}`, `{osm_id}`, etc.) et le texte enrichi dans les descriptions (syntaxe proche du markdown).

### Données distantes vs. stockées

**FAITE :** Toutes les données visibles ne vivent pas dans le stockage uMap. Les calques peuvent référencer des **URL distantes** ou des **requêtes Overpass dynamiques** que le navigateur récupère à l'exécution (tutoriel 11).

**À COMPRENDRE MAINTENANT :** uMap est à la fois un **système de stockage pour du GeoJSON créé** et un **client pour des sources de géodonnées en direct**.

### Fond de carte / couches de tuiles

**FAITE :** Les cartes utilisent des **couches de tuiles** (OSM et alternatives) comme arrière-plan. La couche de tuiles par défaut est intégrée dans les paramètres d'aperçu de la carte via la propriété `preview_settings` du modèle `Map`.

Pensez : **le papier peint derrière vos autocollants** — séparé de vos entités créées.

---

## Ce qui distingue uMap de « simplement consulter une carte »

| Simplement consulter OSM (openstreetmap.org) | uMap |
|---|---|
| Une carte globale de la planète | Un **document cartographique personnalisé** avec sa propre URL |
| Données OSM uniquement | **Fond de carte** OSM + **vos superpositions** |
| Éditer OSM lui-même | Éditer **vos calques** (sans remplacer JOSM/iD) |
| Pas de produit thématique intégrable | **Intégration iframe** avec commandes configurables |
| Pas de permissions par projet | **Statuts de partage/édition** par carte et par calque |
| Pas de pipeline de style pour des superpositions arbitraires | **Style par entité et par calque**, règles conditionnelles, modèles |

**INFÉRENCE :** uMap se situe dans l'espace de la « publication rapide de cartes web » (aux côtés d'outils comme Google My Maps, Carto, etc.) mais est **open source**, **natif OSM** et **auto-hébergeable** — aligné avec l'éthos de la communauté OSM.

---

## Où s'intègre OpenStreetMap

OSM joue **trois rôles distincts** dans uMap. Les garder séparés vous évitera des confusions plus tard.

### 1. Tuiles de fond de carte

**FAITE :** Le README et les tutoriels décrivent des cartes « with OpenStreetMap layers » — les **tuiles raster** dérivées d'OSM constituent l'arrière-plan visuel par défaut.

**Analogie :** Comme utiliser une couche de tuiles Google Maps sous vos points géo Firebase personnalisés — vous voyez les rues et les libellés OSM, mais ce ne sont pas vos données.

### 2. Source de données via Overpass

**FAITE :** Le tutoriel 11 explique comment écrire des requêtes **Overpass QL** (via Overpass Turbo), adapter le format de sortie (`[out:xml]` car uMap comprend le XML OSM, pas le JSON Overpass), et attacher la requête comme **calque de données distant**.

**Analogie :** Comme pointer votre application React vers une API REST en direct au lieu de copier des lignes dans Firestore — la carte se met à jour depuis la base de données live d'OSM.

### 3. Alignement écosystémique, pas édition

**FAITE :** uMap encourage l'*usage* d'OSM (slogan du README). Les assistants d'import tirent des jeux de données d'origine OSM (GeoDataMine, etc.).

**FAITE :** uMap **n'est pas** un éditeur OSM. Vous dessinez/importez *par-dessus* OSM ; éditer OSM lui-même est hors périmètre.

**UTILE PLUS TARD :** Les variables d'entité `osm_type` et `osm_id` (FAQ) apparaissent lorsque les données proviennent d'OSM.

---

## Concepts à maîtriser avant que le code ait du sens

### À COMPRENDRE MAINTENANT

1. **Carte** — un projet cartographique publiable (URL, paramètres, permissions)
2. **Calque de données** — entités regroupées dans une carte ; persistées comme fichier GeoJSON + métadonnées en base
3. **Entité** — un point, une ligne ou un polygone avec propriétés ; géométrie GeoJSON + attributs
4. **Mode édition vs. mode aperçu/consultation** — interface d'auteur vs. interface visiteur
5. **GeoJSON** — le format d'échange pour les données de calque stockées (vue d'ensemble développeur)
6. **Calque de données distant** — données récupérées côté client non stockées dans les fichiers GeoJSON de uMap
7. **Statut de partage / statut d'édition** — modèle d'autorisation pour la consultation et l'édition
8. **Leaflet** — la bibliothèque de rendu cartographique navigateur sur laquelle uMap s'appuie (pas React — **JavaScript vanilla** selon la vue d'ensemble développeur)

### UTILE PLUS TARD (la Phase 2 les enseignera correctement)

- Systèmes de coordonnées, projections, boîtes englobantes (variables `{bbox}` dans la FAQ)
- Pyramides de tuiles et niveaux de zoom
- PostGIS / champs Django GIS (point central sur `Map`)
- Fichiers GeoJSON versionnés (indices de couche de stockage dans `DataLayer.save`)
- Modèles Django, vues, routage d'URL
- Tests d'intégration Playwright, pytest, tests JS Mocha

### Reporter explicitement

- Topologies de déploiement (Docker, Helm, Dokku) — pas nécessaire pour comprendre le produit
- Workflow de traduction (Transifex) — chemin de contribution, pas cœur du produit
- Architecture complète des plugins d'import — jusqu'à ce que nous retracions les flux d'import

---

## Modèle mental compact (gardez-le en tête)

```
┌─────────────────────────────────────────────────────────────┐
│                      Produit uMap                           │
├─────────────────────────────────────────────────────────────┤
│  L'auteur crée une CARTE                                    │
│    ├── paramètres : centre, zoom, fond de carte, légende, UI│
│    ├── permissions : qui peut voir / éditer                 │
│    └── CALQUES DE DONNÉES (ordonnés)                        │
│          ├── Calque A : fichier GeoJSON sur serveur (stocké)│
│          │     └── Entités : points, lignes, polygones      │
│          ├── Calque B : URL distante ou Overpass (fetch live)│
│          └── Calque C : limites admin importées, etc.       │
├─────────────────────────────────────────────────────────────┤
│  Le lecteur ouvre l'URL → Leaflet rend fond + calques       │
│  L'auteur intègre une iframe → même carte, commandes limitées│
└─────────────────────────────────────────────────────────────┘

         OpenStreetMap ──► tuiles de fond de carte
                         └──► données live optionnelles (Overpass)
                         └──► contexte communautaire / mission
```

**Version en une phrase :** uMap est un **outil collaboratif de publication cartographique** où chaque **carte** est une expérience Leaflet configurée sur des tuiles OSM, contenant des **calques** d'**entités GeoJSON** (ou de données distantes), avec **permissions** et **intégration/partage** comme préoccupations de premier ordre.

---

## Mise en garde sur la documentation

**FAITE :** La plupart des tutoriels utilisateur en anglais dans `docs-users/tutorials/` sont des ébauches pointant vers les traductions françaises. Les tutoriels français dans `docs-users/fr/tutorials/` contiennent le contenu utilisateur substantiel. La documentation développeur dans `docs/` est en anglais.

Lorsque nous enseignerons à partir des parcours utilisateur, nous citerons les tutoriels français comme **documentation produit faisant autorité dans ce dépôt**, sans déduire le comportement à partir d'applications cartographiques génériques.

---

## Résumé de la Phase 1 — ce qu'il faut retenir

1. **uMap permet de créer des cartes thématiques** sur des fonds OSM et de les publier/intégrer — ce n'est pas un éditeur OSM.
2. La hiérarchie de contenu est **Carte → Calque de données → Entité (GeoJSON)**.
3. Les données peuvent être **stockées** (fichiers GeoJSON) ou **distantes** (Overpass, URL).
4. La pile est **serveur Django + client vanilla JS/Leaflet** ; la plupart des calculs géo sont côté client.
5. Les permissions (statut de partage/édition) sont une fonctionnalité produit centrale, y compris les cartes anonymes avec liens d'édition secrets.
6. Vos compétences React/TS se transfèrent à la **lecture des patterns d'interface et du flux de données**, mais l'éditeur de carte **n'est pas une application React**.

---

## Ce qui reste incertain (lacunes honnêtes)

| Sujet | Statut |
|---|---|
| Structure exacte des modules frontend | **INCONNU** jusqu'aux Phases 3–5 (la vue d'ensemble développeur dit seulement « vanilla JavaScript ») |
| Quelle part PostGIS est utilisée vs. stockage fichier | **PARTIELLEMENT CONNU** — la vue d'ensemble dit « for the most part » frontend ; `Map.center` utilise `PointField` |
| Si les instances hébergées diffèrent fonctionnellement de l'auto-hébergement | **INFÉRENCE** — probablement des différences de configuration/auth ; pas entièrement documenté en un seul endroit |
| Fournisseurs d'auth par défaut actuels | **INCONNU** — `social-auth-app-django` dans les dépendances suggère OAuth ; nécessite Phase 3/7 |
| Calendrier de couverture des tutoriels anglais | **INCONNU** |

---

## Ce que nous investiguerons ensuite — Phase 2 : Primer domaine

Avant de retracer les chemins de code, nous enseignerons **uniquement les concepts géospatiaux que ce dépôt utilise réellement** :

- Structure GeoJSON telle que uMap la stocke
- Points, lignes, polygones dans l'éditeur
- Calques, style, règles conditionnelles
- Tuiles et fonds de carte (contexte Leaflet)
- Boîtes englobantes et requêtes distantes dynamiques
- OSM / Overpass / XML OSM comme formats de données distantes

Nous éviterons les cours SIG génériques et relierons chaque concept à un fichier, un tutoriel ou un champ de modèle.

---

## Pause ici

La Phase 1 est volontairement centrée sur le produit — pas d'arborescence, pas encore de parcours d'exécution.

**Questions utiles avant la Phase 2 :**

- Quelque chose d'imprécis sur Carte vs. Calque de données vs. Entité ?
- Voulez-vous que la Phase 2 insiste davantage sur les concepts Leaflet (puisque vous connaissez React mais peut-être pas Leaflet) ?
- Prévoyez-vous de contribuer principalement au **frontend (JS)**, au **backend (Django/Python)**, ou **pas encore sûr** ?

Quand vous êtes prêt, dites **« continuer vers la Phase 2 »** (ou posez des questions).

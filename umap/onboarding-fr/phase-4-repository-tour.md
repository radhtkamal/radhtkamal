# Intégration uMap — Phase 4 : Visite du dépôt

> **Statut :** Phase 4 sur 13 · Investigation en lecture seule · S'appuie sur la [Phase 3](phase-3-architecture.md)  
> **Objectif :** Comprendre comment uMap organise son propre code — chemins importants uniquement, avec les relations appelant/appelé et les points de contact pour les contributeurs

---

## Comment utiliser cette visite

La Phase 3 vous a donné **l'architecture**. La Phase 4 vous donne une **carte du code source** — où chercher quand quelque chose casse, et où vos premières PR sont susceptibles d'atterrir.

Chaque section couvre :

- **Ce qu'elle contient**
- **Pourquoi elle existe**
- **Qui l'appelle / ce qu'elle appelle**
- **Données entrantes → données sortantes**
- **Quand vous la modifieriez**

**FAITE :** Cette visite est dérivée du dépôt extrait, et non des conventions génériques Django/Leaflet.

**Note sur la documentation obsolète :** `docs/dev/frontend.md` fait encore référence à `umap.js` et `U.Map`. **FAITE :** Il n'y a pas de `umap.js` dans l'arborescence actuelle. Le point d'entrée client est `umap/static/umap/js/modules/app.js` (classe `App` en export par défaut). Considérez les notes de développement comme utiles mais vérifiez toujours contre les fichiers.

---

## Organisation de haut niveau (ce qui compte vs. ce qu'il faut ignorer)

```
umap/                          ← L'application (Python + JS + templates)
docs/                          ← Documentation développeur et déploiement
docs-users/                    ← Tutoriels utilisateur final (le français est substantiel)
onboarding/                    ← Vos notes d'apprentissage (cette série)
charts/                        ← Chart Helm (ops)
docker/                        ← Configs conteneur/nginx (ops)
scripts/                       ← Sync JS vendors, utilitaires
.github/workflows/             ← CI (docs, helm, hygiène des issues — pas la matrice complète de tests dans cet arbre)
Makefile, pyproject.toml       ← Dépendances Python, commandes test/lint
package.json                   ← Dépendances dev JS (Mocha, Biome, script vendors)
```

| Chemin | Pertinence pour une première contribution |
|---|---|
| `umap/` | **Élevée** — presque tout le code produit |
| `docs/`, `docs-users/` | Moyenne — corrections de docs quand vous comprenez une fonctionnalité |
| `onboarding/` | Vos notes uniquement |
| `charts/`, `docker/` | Faible sauf pour déploiement/infra |
| `scripts/vendorsjs.sh` | Faible sauf pour mettre à jour les bibliothèques embarquées |

---

## Cœur backend (`umap/*.py`)

### `models.py` — vérité du domaine

**Contient :** `Map`, `DataLayer`, `TileLayer`, `Team`, `Star`, `Licence`, `Pictogram`, et les helpers de permissions (`can_edit`, `can_view`, `clone`, corbeille/blocage).

| | |
|---|---|
| **Appelé par** | Vues, formulaires, admin, tests, commandes de gestion |
| **Appelle** | ORM Django ; `settings` ; stockage via le FileField `DataLayer.geojson` |
| **Entrée** | Requêtes HTTP + requêtes ORM |
| **Sortie** | Instances de modèles ; dictionnaires `metadata()` pour le bootstrap client |
| **Vous le modifiez quand** | Ajout de champs carte/couche, règles de permission, comportement de clonage, config en base |

**Conseil contributeur :** Le JSON `settings` client sur `Map` et `DataLayer` reflète les champs `settings` ORM — de nombreuses options UI n'ont jamais leur propre colonne en base.

---

### `views.py` — surface HTTP (~1 500 lignes)

**Contient :** Vues de page (`MapView`, `home`, `search`, tableaux de bord), vues de mutation JSON (`MapCreate`, `MapUpdate`, `DataLayerUpdate`), service de fichiers (`DataLayerView`), `AjaxProxy`, oEmbed, endpoints équipe/utilisateur.

| | |
|---|---|
| **Appelé par** | Routage `urls.py` |
| **Appelle** | `models`, `forms`, `utils.json_dumps`, stockage, `httpx` (proxy) |
| **Entrée** | HTTP GET/POST |
| **Sortie** | Contexte HTML, `simple_json_response()`, octets GeoJSON, redirections |
| **Vous le modifiez quand** | Nouveaux endpoints API, logique de sauvegarde/fusion, téléchargement/export, comportement du proxy |

**Classes clés à garder en favori :**

| Classe | Rôle |
|---|---|
| `MapDetailMixin` | Construit le JSON `map_settings` pour les templates |
| `MapView` | Sert la page carte ; définit `edit_mode` selon les permissions |
| `MapViewGeoJSON` | Même bootstrap en JSON (`/map/{id}/geojson/`) |
| `DataLayerView` | Sert le `.geojson` de la couche (+ gzip, en-tête de version) |
| `DataLayerUpdate` | Sauvegarde la couche ; **fusion en cas de conflit** (HTTP 412) |
| `AjaxProxy` | Récupération d'URL distante mise en cache pour le navigateur |

---

### `urls.py` + `decorators.py` — routage et contrôles d'accès

**Contient :** Toutes les routes nommées ; piles de décorateurs (`can_view_map`, `can_edit_map`, `can_edit_datalayer`, `never_cache` sur les mutations).

| | |
|---|---|
| **Appelé par** | Résolveur d'URL Django |
| **Appelle** | Fonctions/classes de vue après vérification des permissions |
| **Vous le modifiez quand** | Ajout de routes ; renforcement/assouplissement des modèles d'accès |

**FAITE :** `_urls_for_js()` dans `utils.py` introspecte les patterns d'URL **nommés** depuis `umap.urls` (et `umap.sync.app` si le temps réel est activé) et les envoie au client comme modèles d'URI. Si vous ajoutez une route nommée dont le JS a besoin, elle peut apparaître automatiquement — mais le client doit l'appeler via `urls.get('name')`.

---

### `forms.py` — validation côté serveur pour les sauvegardes

**Contient :** `MapSettingsForm`, `DataLayerForm`, formulaires de permissions, `SendLinkForm`.

| | |
|---|---|
| **Appelé par** | Vues `FormLessEditMixin` (POST uniquement) |
| **Entrée** | POST multipart (fichier geojson de couche + chaîne JSON settings) |
| **Sortie** | Champs de modèle nettoyés |
| **Vous le modifiez quand** | Nouveaux champs persistés, règles de validation à la sauvegarde |

**À comparer à :** Zod/Yup sur une API — mais branché sur des ModelForms Django.

---

### `utils.py` — helpers serveur partagés

**Contient :** `_urls_for_js()`, `layers_tree()`, `json_dumps`, `merge_features`, `gzip_file`, `validate_url`, helpers pictogrammes.

| | |
|---|---|
| **Vous le modifiez quand** | Câblage d'URL, forme de l'arborescence des couches pour le client, algorithme de fusion, encodage JSON partagé |

---

### `admin.py` — interface opérateur

**Contient :** Admin GIS pour cartes, fonds de carte, équipes ; actions corbeille/blocage/restauration.

| | |
|---|---|
| **Vous le modifiez quand** | Opérations côté mainteneur (rare pour les nouveaux contributeurs) |

---

### `managers.py` — helpers de queryset

**Contient :** `PublicManager`, `PrivateManager.for_user()` pour les listes page d'accueil/tableau de bord.

| | |
|---|---|
| **Vous le modifiez quand** | Découverte de cartes, filtrage de visibilité dans les vues liste |

---

### `middleware.py`, `context_processors.py`, `autocomplete.py`

| Fichier | Rôle |
|---|---|
| `middleware.py` | Contournement de fuite GEOS ; mode site en lecture seule ; avertissements auth dépréciés |
| `context_processors.py` | Injecte settings/version dans les templates |
| `autocomplete.py` | Recherche utilisateur pour l'assignation d'éditeurs (Agnocomplete) |

**Faible interaction** pour le travail fonctionnel typique sauf si vous travaillez sur l'auth ou la recherche admin.

---

## Persistance (`umap/storage/`)

### `fs.py` — `FSDataStorage`

**Contient :** Chemins GeoJSON versionnés (`datalayer/{…}/{map_id}/{uuid}_{timestamp}.geojson`), purge des anciennes versions, fichiers sidecar gzip.

| | |
|---|---|
| **Appelé par** | Hooks `DataLayer.save()` / `delete()` |
| **Appelé depuis les vues** | `DataLayerView` lit les fichiers ; `DataLayerUpdate` écrit |
| **Vous le modifiez quand** | Disposition des fichiers, politique de rétention, comportement gzip |

### `s3.py` — `S3DataStorage`

Même contrat pour les déploiements en stockage objet.

**FAITE :** `docs/config/storage.md` documente les trois clés `STORAGES` : `default`, `staticfiles`, `data`.

---

## Temps réel (`umap/sync/`)

### `app.py` — synchronisation pair-à-pair WebSocket

**Contient :** Gestionnaire websocket ASGI, salles Redis pub/sub par carte, URL `ws_sync`.

| | |
|---|---|
| **Appelé par** | `asgi.py` quand `scope["type"] == "websocket"` |
| **Appelé depuis le client** | `journal/websocket.js` après `map_websocket_auth_token` |
| **Vous le modifiez quand** | Bugs d'édition collaborative (niche ; `REALTIME_ENABLED` désactivé par défaut) |

---

## Templates (`umap/templates/`)

| Template | Rôle |
|---|---|
| `base.html` | Habillage du site |
| `umap/map_detail.html` | Enveloppe de la page carte ; inclut `map_init.html` |
| `umap/map_init.html` | **div `#map` + JSON `map-settings` + `new App(...)`** |
| `umap/map_fragment.html` | Prévisualisations de cartes embarquées (accueil, listes) |
| `umap/js.html` | Import map, vendors Leaflet, `umap.controls.js`, web components |
| `umap/css.html` | Inclusion des feuilles de style |
| `umap/user_dashboard.html`, `map_table.html` | Liste des cartes de l'utilisateur connecté |

| | |
|---|---|
| **Appelé par** | Vues Django via `template_name` |
| **Appelle** | `{% umap_js %}`, `{% umap_css %}`, `umap_tags.map_fragment` |
| **Vous le modifiez quand** | Nouveaux besoins script/css, changements de forme du bootstrap (peu courant) |

**FAITE :** `templatetags/umap_tags.py` — `map_fragment` utilise `Map.preview_settings` pour les intégrations en liste (plus léger que le bootstrap d'édition complet).

---

## Point d'entrée frontend et ordre de chargement

### Séquence de démarrage

```
map_detail.html
  → js.html (importmap, Leaflet, vendors, umap.controls.js)
  → map_init.html
      → U.SETTINGS = JSON.parse(...)
      → new App("map", U.SETTINGS)
```

### `js/umap.controls.js` — pont de dessin Leaflet hérité

**Contient :** `U.Editable` étendant Leaflet.Editable — événements de dessin, création de marqueurs/lignes/polygones.

| | |
|---|---|
| **Pourquoi un fichier séparé** | Chargé comme script classique (pas module ES) ; étend les globaux `U` et `L` |
| **Appelé depuis** | `LeafletProxy.connectEditTools()` → `new U.Editable(this.app)` |
| **Vous le modifiez quand** | Comportement des outils de dessin, modificateurs clavier pendant l'édition de géométrie |

**FAITE :** La plupart du nouveau code frontend vit dans des modules ES sous `js/modules/`. `umap.controls.js` est la principale exception — l'intégration du dessin précède la modularisation complète.

### `js/modules/global.js`

Exporte `Point`, `LineString`, `Polygon`, `LeafletMarker` sur `window.U` pour les scripts qui ne sont pas encore des modules.

---

## Racine client — `js/modules/app.js`

**Contient :** Classe `App` — init, mode édition, sauvegarde, arborescence des couches, branchement de l'importateur, démarrage du journal, surcharges via query string.

| | |
|---|---|
| **Appelé par** | `map_init.html` uniquement (et les tests) |
| **Appelle** | `LeafletProxy`, `ControlManager`, `DataLayer`, `Journal`, `Formatter`, `URLs`, panneaux/barres |
| **Entrée** | Settings de bootstrap en forme GeoJSON + événements utilisateur |
| **Sortie** | POST HTTP via `ServerRequest` ; mises à jour DOM via `render()` |
| **Vous le modifiez quand** | Comportement éditeur transversal, flux de sauvegarde, nouveaux raccourcis globaux, paramètres d'intégration en query string |

**C'est le hub.** Quand vous ne savez pas où vit la logique, commencez ici et grep vers l'extérieur.

---

## Câblage des URL — `js/modules/urls.js`

Encapsule les modèles d'URI fournis par le serveur.

| Méthode | Comportement |
|---|---|
| `get('datalayer_view', { map_id, pk })` | Remplace `{map_id}`, `{pk}` |
| `map_save({ map_id })` | URL de création vs mise à jour de carte |
| `datalayer_save({ created, ... })` | **Nommage inversé :** `created: true` → `datalayer_update` ; `false` → `datalayer_create` |

**FAITE :** Le nommage de `datalayer_save` se prête facilement à une mauvaise lecture — `created` signifie « existe déjà sur le serveur » (chemin de mise à jour), pas « en cours de création maintenant ».

---

## Couche de données — `js/modules/data/`

### `layer.js` — `DataLayer` client

**Contient :** Collection de features, récupération distante, sauvegarde (`FormData` + blob geojson), type de style, règles, filtres, légende.

| | |
|---|---|
| **Appelé par** | `App.createDataLayer`, updaters du journal |
| **Appelle** | GET `datalayer_view` ; POST `datalayer_save` ; `Formatter` ; couches de rendu |
| **Vous le modifiez quand** | Chargement/sauvegarde de couche, données distantes, effets de bord de l'UI des paramètres de couche |

### `features.js` — `Point`, `LineString`, `Polygon`

**Contient :** Propriétés par feature, templates de popup, panneaux d'édition, mesure, métadonnées du journal.

| | |
|---|---|
| **Vous le modifiez quand** | Éditeur de feature, contenu de popup, gestion des propriétés, fin de dessin |

### `fields.js` — champs de formulaire dynamiques pour les propriétés

Widgets de champs pilotés par schéma pour le panneau d'édition.

### `types.js` — Choropleth, Cluster, Heat, etc.

Calculs de type de visualisation (classes, couleurs, légende).

| | |
|---|---|
| **Vous le modifiez quand** | Modes de style, cercles proportionnels, algorithmes de classification |

---

## Schéma et formulaires — `schema.js` + `form/`

### `schema.js`

**Contient :** `SCHEMA` — chaque propriété carte/couche : type, `impacts` (`ui`, `data`, `remote-data`, …), labels, valeurs par défaut.

| | |
|---|---|
| **Appelé par** | `form/builder.js` (`MutatingForm`), updaters du journal, validation |
| **Vous le modifiez quand** | **Ajout de tout nouveau paramètre carte/couche** — c'est le registre |

**À COMPRENDRE MAINTENANT pour le travail sur les paramètres :** Nouvelle propriété = entrée de schéma + généralement form builder + parfois impact `MapUpdater`/`DataLayerUpdater`.

### `form/builder.js`, `form/fields.js`

Construisent les panneaux d'édition depuis le schéma. Respectent les patterns UX de formulaire uMap (compteurs de fieldset, entrées d'aide).

---

## Journal — `js/modules/journal/`

| Fichier | Rôle |
|---|---|
| `engine.js` | Journal des opérations, `save()`, annuler/rétablir, dispatch de sync pair |
| `updaters.js` | Applique les opérations distantes/locales à `App`, `DataLayer`, features, permissions |
| `undo.js` | Pile d'annulation |
| `websocket.js` | Transport vers `ws_sync` |
| `hlc.js` | Horloge logique hybride pour l'ordonnancement |

| | |
|---|---|
| **Vous le modifiez quand** | Bugs d'ordonnancement de sauvegarde, annuler/rétablir, sync collaborative (avancé) |

**FAITE :** La formulation « dirty » de `docs/dev/frontend.md` reste exacte — le journal suit ce qui doit être persisté.

---

## Rendu — `js/modules/rendering/`

| Fichier | Rôle |
|---|---|
| `leaflet.js` | `LeafletProxy` — carte, tuiles, features, événements, contexte bbox |
| `openlayers.js` | `OLProxy` expérimental (`?openlayers`) |
| `ui.js` | Classes de couches Leaflet, marqueurs, style des tracés |
| `layers/base.js`, `cluster.js`, `heat.js` | Renderers par type de couche |
| `template.js` | HTML popup/infobulle depuis les templates de feature |

| | |
|---|---|
| **Vous le modifiez quand** | Bugs d'affichage carte, comportement cluster/heat, icônes de marqueurs, poignées d'édition |

**Frontière :** Le rendu lit le GeoJSON ; il ne doit pas faire de POST vers le serveur.

---

## Enveloppe UI — `js/modules/ui/`

| Fichier | Rôle |
|---|---|
| `controls.js` | `ControlManager` — zoom, localisation, navigation, intégration, etc. |
| `panel.js` | Panneaux latéraux (édition, plein écran, navigation) |
| `bar.js` | Barres d'outils haut/bas/édition |
| `dialog.js`, `tooltip.js`, `loader.js` | Modales, infobulles, indicateur de chargement |
| `hash.js` | Hash d'URL ↔ centre/zoom de la carte |

| | |
|---|---|
| **Vous le modifiez quand** | Nouveau contrôle carte, bouton de barre d'outils, disposition de panneau |

---

## Import / export

| Fichier | Rôle |
|---|---|
| `importer.js` | Dialogue d'import ; fichier/URL/collage ; dispatch vers les helpers |
| `importers/*.js` | Helpers configurés serveur (`UMAP_IMPORTERS`) |
| `formatter.js` | Conversion de format (gpx, kml, osm, …) |
| `share.js` | Panneau d'export, snippet iframe, export image |

| | |
|---|---|
| **Config** | `docs/config/importers.md`, `UMAP_IMPORTERS` dans settings |
| **Vous le modifiez quand** | Nouvelle source d'import, support de format, option d'export |

**FAITE :** `importer.js` utilise un `switch (name)` explicite pour les imports dynamiques — ajouter un helper nécessite **à la fois** la config settings **et** une nouvelle branche case (ou extension du switch).

---

## Modules client transversaux

| Module | Rôle | À toucher quand |
|---|---|---|
| `permissions.js` | UI et sauvegarde des permissions carte + couche | UX de contrôle d'accès |
| `rules.js` | Règles de style conditionnel | Syntaxe/évaluation des règles |
| `filters.js` | Filtres du navigateur de données | UX de filtrage |
| `browser.js` | UI tableau « Parcourir les données » | Fonctionnalités du tableau de données |
| `autocomplete.js` | Boîte de recherche, recherche de lieu | Recherche de géocodage |
| `request.js` | Wrapper `fetch`, CSRF, alertes d'erreur | Comportement du client HTTP |
| `geoutils.js` | Wrappers Turf (bbox, mesure, …) | Helpers de géométrie |
| `caption.js` | Panneau légende / à propos | Contenu de légende |
| `slideshow.js` | Mode diaporama | Fonctionnalité diaporama |
| `i18n.js` + `locale/*.js` | Traductions | Chaînes UI (souvent via Transifex) |

---

## Web components — `js/components/`

| Fichier | Rôle |
|---|---|
| `alerts/alert.js` | UI toast/alerte (`Alert`, `AlertConflict`) |
| `fragment.js`, `modal.js`, `copiable.js` | Composants DOM réutilisables |

Chargés depuis `js.html` en tant que modules. Utilisez-les plutôt que d'inventer de nouveaux patterns d'alerte.

---

## Vendors — `static/umap/vendors/`

Plugins Leaflet embarqués, sous-ensembles Turf, osm2geojson, csv2geojson, etc.

| | |
|---|---|
| **Mis à jour via** | `npm run vendors` / `scripts/vendorsjs.sh` |
| **Vous le modifiez quand** | Mise à niveau des bibliothèques — pas pour la logique fonctionnelle |

**Ne modifiez pas les fichiers vendor pour le travail fonctionnel.**

---

## Tests

### Python — `umap/tests/`

| Domaine | Exemples |
|---|---|
| Modèles/stockage | `test_datalayer.py`, `test_datalayer_s3.py`, `test_merge_features.py` |
| Vues/API | `test_map_views.py`, `test_datalayer_views.py` |
| Commandes | `test_clean_tilelayer.py`, `test_purge_old_versions.py` |
| Fixtures/factories | `base.py` (`MapFactory`, `DataLayerFactory`) |

**Exécution :** `make test-unit` ou `pytest umap/tests/ --ignore umap/tests/integration`

### Playwright — `umap/tests/integration/`

Un fichier par préoccupation utilisateur : `test_save.py`, `test_import.py`, `test_remote_data.py`, `test_choropleth.py`, …

**Exécution :** `make test-integration` — nécessite une app en cours d'exécution + navigateur.

**Pattern contributeur :** Bug de dessin → `test_draw_*.py` ; bug de sauvegarde → `test_save.py` ou `test_datalayer_views.py`.

### Tests unitaires JS — `umap/static/umap/unittests/`

Tests Mocha pour `geoutils`, `schema`, URLs, etc.

**Exécution :** `make testjs`

---

## Documentation et configuration

| Chemin | Public |
|---|---|
| `docs/install.md` | Configuration locale |
| `docs/contributing.md` | Tests, lint, attentes PR |
| `docs/dev/frontend.md` | Notes client (noms partiellement obsolètes) |
| `docs/config/settings.md` | Tous les settings `UMAP_*` |
| `docs/config/storage.md`, `importers.md` | Config stockage et import |
| `docs-users/fr/tutorials/` | Référence de parcours utilisateur réel |

---

## Conventions de nommage et d'organisation (audit du dépôt)

Comment uMap pense son code — **patterns FAITE issus de l'arborescence :**

### Python

| Pattern | Exemple |
|---|---|
| Package unique d'app Django | `umap/` |
| Modules plats pour les préoccupations majeures | `views.py`, `models.py`, `forms.py` |
| Noms d'URL | `snake_case` : `map_update`, `datalayer_view` |
| Vues basées sur des classes | Génériques Django + mixins (`MapDetailMixin`) |
| Réponses JSON | `simple_json_response(**kwargs)` — pas DRF |
| Décorateurs de permission | `can_edit_map`, `can_view_map` sur les patterns d'URL |
| Commandes de gestion | `umap/management/commands/*.py` |

### JavaScript

| Pattern | Exemple |
|---|---|
| Modules ES dans `js/modules/` | `import { DataLayer } from './data/layer.js'` |
| Export par défaut pour l'entrée app | `export default class App` |
| Classes PascalCase | `LeafletProxy`, `ControlManager`, `Journal` |
| Bus d'événements | `Utils.WithEvents` → `fire('datalayer:changed')` |
| Pont global hérité | `window.U` via `global.js` + `umap.controls.js` |
| Paramètres pilotés par schéma | `schema.js` + `form/builder.js` |
| Enregistrement explicite des importateurs | `switch` dans `importer.js` par nom de helper |
| Tests regroupés par domaine | `unittests/geoutils.js`, intégration par fonctionnalité |

### Templates et statiques

| Pattern | Exemple |
|---|---|
| Templates d'app sous `templates/umap/` | |
| Statiques sous `static/umap/` | `js/`, `css/`, `img/`, `locale/`, `vendors/` |
| Stockage staticfiles avec hash | `UmapManifestStaticFilesStorage` |
| i18n | Django `.po` + JS `locale/{lang}.js` |

### Tests

| Pattern | Exemple |
|---|---|
| `test_<area>.py` | pytest |
| `test_<feature>.py` dans `integration/` | Playwright |
| Factories dans `tests/base.py` | `MapFactory`, `DataLayerFactory` |

### Ce que uMap **ne fait pas**

- Pas de layout monorepo `src/`
- Pas de frontend React/Vue
- Pas de Django REST Framework
- Pas de packages Python par fonctionnalité — les fonctionnalités sont des fichiers transversaux

---

## Aide-mémoire du graphe d'appels

### Ouvrir une carte

```
urls.py → MapView
  → models.Map + datalayers.metadata()
  → map_detail.html → map_init.html
  → App.init()
  → constructeur DataLayer (métadonnées uniquement)
  → DataLayer.fetchData() → GET datalayer_view
  → LeafletProxy.addFeature(...)
```

### Sauvegarder des modifications

```
Utilisateur Ctrl+S
  → App.saveAll()
  → Journal.save()
  → DataLayer.save() / Map.save() (méthodes client)
  → POST datalayer_update / map_update
  → forms → models → FSDataStorage nouvelle version .geojson
```

### Importer une couche Overpass distante

```
Dialogue Importateur → importers/overpass.js
  → définit remoteData.url + format=osm
  → DataLayer.fetchRemoteData()
  → ajax_proxy optionnel
  → Formatter.parse → fromGeoJSON
```

---

## Où atterrissent généralement les premières contributions

| Si vous travaillez sur… | Commencez dans… | Vérifiez aussi… |
|---|---|---|
| Contrôle carte / barre d'outils | `ui/controls.js`, `ui/bar.js` | `schema.js` si nouveau paramètre |
| Panneau d'édition / paramètres carte | `form/builder.js`, `schema.js` | Vue `MapUpdate` |
| Dessin de marqueurs/lignes | `umap.controls.js`, `features.js` | Playwright `test_draw_*` |
| Bugs chargement/sauvegarde de couche | `data/layer.js`, `views.py` DataLayer* | `storage/fs.py`, `test_datalayer_views.py` |
| Format d'import | `formatter.js` | `test_import.py` |
| Nouveau helper d'import | `importers/yours.js`, switch `importer.js` | `docs/config/importers.md` |
| UX permissions | `permissions.js` | `decorators.py`, `models.can_edit` |
| Variables popup/template | `features.js`, `templates.js` | FAQ dans docs-users |
| Données distantes / proxy | `data/layer.js`, `AjaxProxy` | `test_remote_data.py` |
| Choropleth/cluster | `data/types.js`, `rendering/layers/` | `test_choropleth.py`, `test_cluster.py` |
| Recherche backend/page d'accueil | `views.py`, `managers.py` | `test_map_views.py` |

### Sûr de reporter

| Domaine | Pourquoi |
|---|---|
| `sync/` + Redis | Fonctionnalité optionnelle, état distribué complexe |
| `openlayers.js` | Migration en cours |
| `charts/`, `docker/` | Ops, pas logique produit |
| La plupart de `management/commands/` | Outils admin/maintenance |

---

## Résumé de la Phase 4

### À MÉMORISER

1. **`app.js` est le hub client** — pas `umap.js` (docs obsolètes).
2. **`schema.js` enregistre les paramètres** — les nouvelles options commencent ici.
3. **`views.py` + `data/layer.js`** constituent le contrat de chargement/sauvegarde.
4. **`umap.controls.js` est le pont des outils de dessin** — script global hérité.
5. **Les tests reflètent les fonctionnalités** — `integration/test_<feature>.py` est votre filet de sécurité.

### UTILE PLUS TARD

- Internes de `journal/`
- `management/commands/`
- Chemins Helm/Docker

### À IGNORER POUR L'INSTANT

- Arborescence complète `vendors/`
- Pipeline de traduction (`tx pull`, `compilemessages`) jusqu'à faire de l'i18n
- `charts/umap` sauf pour déployer

---

## Ce qui reste incertain

| Sujet | Statut |
|---|---|
| Emplacement du workflow CI de tests complet | **PARTIELLEMENT CONNU** — le `Makefile` définit les tests ; `.github/workflows/` ici est surtout docs/helm — l'upstream peut exécuter les tests ailleurs |
| Si toutes les instances activent les mêmes `UMAP_IMPORTERS` | **INCONNU** — spécifique au déploiement |
| Liste complète des paramètres d'intégration en query string | **INFÉRENCE** — dispersés dans `app.setPropertiesFromQueryString()` — Phase 5 |

---

## Ce que nous investiguons ensuite — Phase 5 : Parcours verticaux d'exécution

Nous choisissons 2 à 4 actions utilisateur réelles et les traçons de bout en bout avec les fonctions exactes :

1. **Ouvrir une carte existante** (probablement en premier — chemin de chargement)
2. **Dessiner un marqueur et sauvegarder**
3. **Importer ou récupérer une couche distante**
4. **Modifier les permissions ou l'intégration**

Chaque flux reçoit un diagramme Mermaid et une narration pas à pas.

**Recommandation par défaut :** Commencer par **ouvrir une carte** + **sauvegarder un marqueur** sauf si vous préférez un ordre orienté frontend ou backend.

---

## Pause ici

Vous devriez maintenant savoir **où chercher** sans une liste complète de l'arborescence.

**Questions avant la Phase 5 :**

- Ordre de parcours orienté frontend ou backend ?
- Un module de cette visite qui reste opaque (`journal`, `schema`, nommage `datalayer_save`) ?
- Voulez-vous que le premier parcours inclue les **payloads réseau** (forme FormData) ?

Quand vous êtes prêt, dites **« continuer vers la Phase 5 »** ou nommez le flux à tracer en premier.

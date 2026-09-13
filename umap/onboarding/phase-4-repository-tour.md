# uMap Onboarding — Phase 4: Repository Tour

> **Status:** Phase 4 of 13 · Read-only investigation · Builds on [Phase 3](phase-3-architecture.md)  
> **Goal:** Learn how uMap organizes its own code — important paths only, with caller/callee relationships and contributor touchpoints

---

## How to use this tour

Phase 3 gave you the **architecture**. Phase 4 gives you a **map of the codebase** — where to look when something breaks, and where your first PRs are likely to land.

Each section covers:

- **What it contains**
- **Why it exists**
- **Who calls it / what it calls**
- **Data in → data out**
- **When you would modify it**

**FACT:** This tour is derived from the checked-out repository, not from generic Django/Leaflet conventions.

**Note on outdated docs:** `docs/dev/frontend.md` still refers to `umap.js` and `U.Map`. **FACT:** There is no `umap.js` in the current tree. The client entry point is `umap/static/umap/js/modules/app.js` (default-export class `App`). Treat dev notes as helpful but verify against files.

---

## Top-level layout (what matters vs. what to skip)

```
umap/                          ← THE application (Python + JS + templates)
docs/                          ← Developer & deploy documentation
docs-users/                    ← End-user tutorials (French is substantive)
onboarding/                    ← Your learning notes (this series)
charts/                        ← Helm chart (ops)
docker/                        ← Container/nginx configs (ops)
scripts/                       ← Vendor JS sync, utilities
.github/workflows/             ← CI (docs, helm, issue hygiene — not full test matrix in this tree)
Makefile, pyproject.toml       ← Python deps, test/lint commands
package.json                   ← JS dev deps (Mocha, Biome, vendors script)
```

| Path | First-contribution relevance |
|---|---|
| `umap/` | **High** — almost all product code |
| `docs/`, `docs-users/` | Medium — docs fixes when you understand a feature |
| `onboarding/` | Your notes only |
| `charts/`, `docker/` | Low unless doing deploy/infra |
| `scripts/vendorsjs.sh` | Low unless upgrading vendored libraries |

---

## Backend core (`umap/*.py`)

### `models.py` — domain truth

**Contains:** `Map`, `DataLayer`, `TileLayer`, `Team`, `Star`, `Licence`, `Pictogram`, and permission helpers (`can_edit`, `can_view`, `clone`, trash/block).

| | |
|---|---|
| **Called by** | Views, forms, admin, tests, management commands |
| **Calls** | Django ORM; `settings`; storage via `DataLayer.geojson` FileField |
| **In** | HTTP requests + ORM queries |
| **Out** | Model instances; `metadata()` dicts for client bootstrap |
| **You change it when** | Adding map/layer fields, permission rules, clone behavior, DB-backed config |

**Contributor tip:** Client `settings` JSON on `Map` and `DataLayer` mirrors ORM `settings` fields — many UI options never get their own DB column.

---

### `views.py` — HTTP surface (~1,500 lines)

**Contains:** Page views (`MapView`, `home`, `search`, dashboards), JSON mutation views (`MapCreate`, `MapUpdate`, `DataLayerUpdate`), file serving (`DataLayerView`), `AjaxProxy`, oEmbed, team/user endpoints.

| | |
|---|---|
| **Called by** | `urls.py` routing |
| **Calls** | `models`, `forms`, `utils.json_dumps`, storage, `httpx` (proxy) |
| **In** | HTTP GET/POST |
| **Out** | HTML context, `simple_json_response()`, GeoJSON bytes, redirects |
| **You change it when** | New API endpoints, save/merge logic, download/export, proxy behavior |

**Key classes to bookmark:**

| Class | Role |
|---|---|
| `MapDetailMixin` | Builds `map_settings` JSON for templates |
| `MapView` | Serves map page; sets `edit_mode` from permissions |
| `MapViewGeoJSON` | Same bootstrap as JSON (`/map/{id}/geojson/`) |
| `DataLayerView` | Serves layer `.geojson` (+ gzip, version header) |
| `DataLayerUpdate` | Saves layer; **merge on conflict** (HTTP 412) |
| `AjaxProxy` | Cached remote URL fetch for browser |

---

### `urls.py` + `decorators.py` — routing & gates

**Contains:** All named routes; decorator stacks (`can_view_map`, `can_edit_map`, `can_edit_datalayer`, `never_cache` on mutations).

| | |
|---|---|
| **Called by** | Django URL resolver |
| **Calls** | View functions/classes after permission checks |
| **You change it when** | Adding routes; tightening/loosening access patterns |

**FACT:** `_urls_for_js()` in `utils.py` introspects **named** URL patterns from `umap.urls` (and `umap.sync.app` if realtime enabled) and ships them to the client as URI templates. If you add a named route the JS needs, it may appear automatically — but the client must call it via `urls.get('name')`.

---

### `forms.py` — server-side validation for saves

**Contains:** `MapSettingsForm`, `DataLayerForm`, permission forms, `SendLinkForm`.

| | |
|---|---|
| **Called by** | `FormLessEditMixin` views (POST only) |
| **In** | Multipart POST (layer geojson file + settings JSON string) |
| **Out** | Cleaned model fields |
| **You change it when** | New persisted fields, validation rules on save |

**Compare to:** Zod/Yup on an API — but wired to Django ModelForms.

---

### `utils.py` — shared server helpers

**Contains:** `_urls_for_js()`, `layers_tree()`, `json_dumps`, `merge_features`, `gzip_file`, `validate_url`, pictogram helpers.

| | |
|---|---|
| **You change it when** | URL wiring, datalayer tree shape for client, merge algorithm, shared JSON encoding |

---

### `admin.py` — operator UI

**Contains:** GIS admin for maps, tile layers, teams; trash/block/restore actions.

| | |
|---|---|
| **You change it when** | Maintainer-facing ops (rare for new contributors) |

---

### `managers.py` — queryset helpers

**Contains:** `PublicManager`, `PrivateManager.for_user()` for homepage/dashboard listings.

| | |
|---|---|
| **You change it when** | Map discovery, visibility filtering in list views |

---

### `middleware.py`, `context_processors.py`, `autocomplete.py`

| File | Role |
|---|---|
| `middleware.py` | GEOS leak workaround; readonly site mode; deprecated auth warnings |
| `context_processors.py` | Injects settings/version into templates |
| `autocomplete.py` | User search for editor assignment (Agnocomplete) |

**Low touch** for typical feature work unless you work on auth or admin search.

---

## Persistence (`umap/storage/`)

### `fs.py` — `FSDataStorage`

**Contains:** Versioned GeoJSON paths (`datalayer/{…}/{map_id}/{uuid}_{timestamp}.geojson`), purge old versions, gzip sidecar files.

| | |
|---|---|
| **Called by** | `DataLayer.save()` / `delete()` hooks |
| **Called from views** | `DataLayerView` reads files; `DataLayerUpdate` writes |
| **You change it when** | File layout, retention policy, gzip behavior |

### `s3.py` — `S3DataStorage`

Same contract for object storage deployments.

**FACT:** `docs/config/storage.md` documents the three `STORAGES` keys: `default`, `staticfiles`, `data`.

---

## Realtime (`umap/sync/`)

### `app.py` — WebSocket peer sync

**Contains:** ASGI websocket handler, Redis pub/sub rooms per map, `ws_sync` URL.

| | |
|---|---|
| **Called by** | `asgi.py` when `scope["type"] == "websocket"` |
| **Called from client** | `journal/websocket.js` after `map_websocket_auth_token` |
| **You change it when** | Collaborative editing bugs (niche; `REALTIME_ENABLED` defaults off) |

---

## Templates (`umap/templates/`)

| Template | Role |
|---|---|
| `base.html` | Site chrome |
| `umap/map_detail.html` | Map page shell; includes `map_init.html` |
| `umap/map_init.html` | **`#map` div + `map-settings` JSON + `new App(...)`** |
| `umap/map_fragment.html` | Embedded map previews (homepage, lists) |
| `umap/js.html` | Import map, Leaflet vendors, `umap.controls.js`, web components |
| `umap/css.html` | Stylesheet inclusion |
| `umap/user_dashboard.html`, `map_table.html` | Logged-in user map list |

| | |
|---|---|
| **Called by** | Django views via `template_name` |
| **Calls into** | `{% umap_js %}`, `{% umap_css %}`, `umap_tags.map_fragment` |
| **You change it when** | New script/css needs, bootstrap shape changes (uncommon) |

**FACT:** `templatetags/umap_tags.py` — `map_fragment` uses `Map.preview_settings` for list embeds (lighter than full edit bootstrap).

---

## Frontend entry & loading order

### Boot sequence

```
map_detail.html
  → js.html (importmap, Leaflet, vendors, umap.controls.js)
  → map_init.html
      → U.SETTINGS = JSON.parse(...)
      → new App("map", U.SETTINGS)
```

### `js/umap.controls.js` — legacy Leaflet drawing bridge

**Contains:** `U.Editable` extending Leaflet.Editable — drawing events, marker/line/polygon creation.

| | |
|---|---|
| **Why separate file** | Loaded as classic script (not ES module); extends global `U` and `L` |
| **Called from** | `LeafletProxy.connectEditTools()` → `new U.Editable(this.app)` |
| **You change it when** | Draw tool behavior, keyboard modifiers during geometry edit |

**FACT:** Most new frontend code lives in ES modules under `js/modules/`. `umap.controls.js` is the main exception — drawing integration predating full modularization.

### `js/modules/global.js`

Exports `Point`, `LineString`, `Polygon`, `LeafletMarker` onto `window.U` for scripts that are not yet modules.

---

## Client root — `js/modules/app.js`

**Contains:** `App` class — init, edit mode, save, layer tree, importer hookup, journal startup, query-string overrides.

| | |
|---|---|
| **Called by** | `map_init.html` only (and tests) |
| **Calls** | `LeafletProxy`, `ControlManager`, `DataLayer`, `Journal`, `Formatter`, `URLs`, panels/bars |
| **In** | Bootstrap GeoJSON-shaped settings + user events |
| **Out** | HTTP POSTs via `ServerRequest`; DOM updates via `render()` |
| **You change it when** | Cross-cutting editor behavior, save flow, new global shortcuts, embed query params |

**This is the hub.** When you do not know where logic lives, start here and grep outward.

---

## URL wiring — `js/modules/urls.js`

Wraps server-provided URI templates.

| Method | Behavior |
|---|---|
| `get('datalayer_view', { map_id, pk })` | Substitutes `{map_id}`, `{pk}` |
| `map_save({ map_id })` | Create vs update map URL |
| `datalayer_save({ created, ... })` | **Inverted naming:** `created: true` → `datalayer_update`; `false` → `datalayer_create` |

**FACT:** The `datalayer_save` naming is easy to misread — `created` means "already exists on server" (update path), not "being created now".

---

## Data layer — `js/modules/data/`

### `layer.js` — client `DataLayer`

**Contains:** Feature collection, remote fetch, save (`FormData` + geojson blob), styling type, rules, filters, legend.

| | |
|---|---|
| **Called by** | `App.createDataLayer`, journal updaters |
| **Calls** | `datalayer_view` GET; `datalayer_save` POST; `Formatter`; rendering layers |
| **You change it when** | Layer load/save, remote data, layer settings UI side effects |

### `features.js` — `Point`, `LineString`, `Polygon`

**Contains:** Per-feature properties, popup templates, edit panels, measure, journal metadata.

| | |
|---|---|
| **You change it when** | Feature editor, popup content, property handling, draw completion |

### `fields.js` — dynamic form fields for properties

Schema-driven field widgets for the edit panel.

### `types.js` — Choropleth, Cluster, Heat, etc.

Visualization type computations (breaks, colors, legend).

| | |
|---|---|
| **You change it when** | Styling modes, proportional circles, classification algorithms |

---

## Schema & forms — `schema.js` + `form/`

### `schema.js`

**Contains:** `SCHEMA` — every map/layer property: type, `impacts` (`ui`, `data`, `remote-data`, …), labels, defaults.

| | |
|---|---|
| **Called by** | `form/builder.js` (`MutatingForm`), journal updaters, validation |
| **You change it when** | **Adding any new map/layer setting** — this is the registry |

**MUST UNDERSTAND NOW for settings work:** New property = schema entry + usually form builder + sometimes `MapUpdater`/`DataLayerUpdater` impact.

### `form/builder.js`, `form/fields.js`

Build edit panels from schema. Match uMap's form UX patterns (fieldset counters, help entries).

---

## Journal — `js/modules/journal/`

| File | Role |
|---|---|
| `engine.js` | Operation log, `save()`, undo/redo, peer sync dispatch |
| `updaters.js` | Apply remote/local ops to `App`, `DataLayer`, features, permissions |
| `undo.js` | Undo stack |
| `websocket.js` | Transport to `ws_sync` |
| `hlc.js` | Hybrid logical clock for ordering |

| | |
|---|---|
| **You change it when** | Save ordering bugs, undo/redo, collaborative sync (advanced) |

**FACT:** `docs/dev/frontend.md` "dirty" wording is still accurate — journal tracks what needs persisting.

---

## Rendering — `js/modules/rendering/`

| File | Role |
|---|---|
| `leaflet.js` | `LeafletProxy` — map, tiles, features, events, bbox context |
| `openlayers.js` | Experimental `OLProxy` (`?openlayers`) |
| `ui.js` | Leaflet layer classes, markers, path styling |
| `layers/base.js`, `cluster.js`, `heat.js` | Layer type renderers |
| `template.js` | Popup/tooltip HTML from feature templates |

| | |
|---|---|
| **You change it when** | Map display bugs, cluster/heat behavior, marker icons, edit handles |

**Boundary:** Rendering reads GeoJSON; it should not POST to server.

---

## UI shell — `js/modules/ui/`

| File | Role |
|---|---|
| `controls.js` | `ControlManager` — zoom, locate, browse, embed, etc. |
| `panel.js` | Side panels (edit, full, browse) |
| `bar.js` | Top/bottom/edit toolbars |
| `dialog.js`, `tooltip.js`, `loader.js` | Modals, hover tips, loading indicator |
| `hash.js` | URL hash ↔ map center/zoom |

| | |
|---|---|
| **You change it when** | New map control, toolbar button, panel layout |

---

## Import / export

| File | Role |
|---|---|
| `importer.js` | Import dialog; file/URL/paste; dispatches to helpers |
| `importers/*.js` | Server-configured helpers (`UMAP_IMPORTERS`) |
| `formatter.js` | Format conversion (gpx, kml, osm, …) |
| `share.js` | Export panel, iframe snippet, image export |

| | |
|---|---|
| **Config** | `docs/config/importers.md`, `UMAP_IMPORTERS` in settings |
| **You change it when** | New import source, format support, export option |

**FACT:** `importer.js` uses explicit `switch (name)` for dynamic imports — adding a helper requires **both** settings config **and** a new case branch (or extending the switch).

---

## Cross-cutting client modules

| Module | Role | Touch when |
|---|---|---|
| `permissions.js` | Map + layer permission UI and save | Access control UX |
| `rules.js` | Conditional styling rules | Rule syntax/evaluation |
| `filters.js` | Data browser filters | Filtering UX |
| `browser.js` | "Browse data" table UI | Data table features |
| `autocomplete.js` | Search box, place lookup | Geocoding search |
| `request.js` | `fetch` wrapper, CSRF, error alerts | HTTP client behavior |
| `geoutils.js` | Turf wrappers (bbox, measure, …) | Geometry helpers |
| `caption.js` | Legend / about panel | Caption content |
| `slideshow.js` | Slideshow mode | Slideshow feature |
| `i18n.js` + `locale/*.js` | Translations | UI strings (often via Transifex) |

---

## Web components — `js/components/`

| File | Role |
|---|---|
| `alerts/alert.js` | Toast/alert UI (`Alert`, `AlertConflict`) |
| `fragment.js`, `modal.js`, `copiable.js` | Reusable DOM components |

Loaded from `js.html` as modules. Use these instead of inventing new alert patterns.

---

## Vendors — `static/umap/vendors/`

Vendored Leaflet plugins, Turf subsets, osm2geojson, csv2geojson, etc.

| | |
|---|---|
| **Updated via** | `npm run vendors` / `scripts/vendorsjs.sh` |
| **You change it when** | Upgrading libraries — not for feature logic |

**Do not edit vendor files for feature work.**

---

## Tests

### Python — `umap/tests/`

| Area | Examples |
|---|---|
| Models/storage | `test_datalayer.py`, `test_datalayer_s3.py`, `test_merge_features.py` |
| Views/API | `test_map_views.py`, `test_datalayer_views.py` |
| Commands | `test_clean_tilelayer.py`, `test_purge_old_versions.py` |
| Fixtures/factories | `base.py` (`MapFactory`, `DataLayerFactory`) |

**Run:** `make test-unit` or `pytest umap/tests/ --ignore umap/tests/integration`

### Playwright — `umap/tests/integration/`

One file per user-facing concern: `test_save.py`, `test_import.py`, `test_remote_data.py`, `test_choropleth.py`, …

**Run:** `make test-integration` — needs running app + browser.

**Contributor pattern:** Bug in draw → `test_draw_*.py`; bug in save → `test_save.py` or `test_datalayer_views.py`.

### JS unit — `umap/static/umap/unittests/`

Mocha tests for `geoutils`, `schema`, URLs, etc.

**Run:** `make testjs`

---

## Documentation & config

| Path | Audience |
|---|---|
| `docs/install.md` | Local setup |
| `docs/contributing.md` | Tests, lint, PR expectations |
| `docs/dev/frontend.md` | Client notes (partially stale names) |
| `docs/config/settings.md` | All `UMAP_*` settings |
| `docs/config/storage.md`, `importers.md` | Storage & import config |
| `docs-users/fr/tutorials/` | Real user-journey reference |

---

## Naming & organization conventions (repository audit)

How uMap thinks about its code — **FACT patterns from the tree:**

### Python

| Pattern | Example |
|---|---|
| Single Django app package | `umap/` |
| Flat modules for major concerns | `views.py`, `models.py`, `forms.py` |
| URL names | `snake_case`: `map_update`, `datalayer_view` |
| Class-based views | Django generics + mixins (`MapDetailMixin`) |
| JSON responses | `simple_json_response(**kwargs)` — not DRF |
| Permission decorators | `can_edit_map`, `can_view_map` on URL patterns |
| Management commands | `umap/management/commands/*.py` |

### JavaScript

| Pattern | Example |
|---|---|
| ES modules in `js/modules/` | `import { DataLayer } from './data/layer.js'` |
| Default export for app entry | `export default class App` |
| PascalCase classes | `LeafletProxy`, `ControlManager`, `Journal` |
| Event bus | `Utils.WithEvents` → `fire('datalayer:changed')` |
| Legacy global bridge | `window.U` via `global.js` + `umap.controls.js` |
| Schema-driven settings | `schema.js` + `form/builder.js` |
| Explicit importer registration | `switch` in `importer.js` per helper name |
| Tests colocated by domain | `unittests/geoutils.js`, integration by feature |

### Templates & static

| Pattern | Example |
|---|---|
| App templates under `templates/umap/` | |
| Static under `static/umap/` | `js/`, `css/`, `img/`, `locale/`, `vendors/` |
| Hashed staticfiles storage | `UmapManifestStaticFilesStorage` |
| i18n | Django `.po` + JS `locale/{lang}.js` |

### Tests

| Pattern | Example |
|---|---|
| `test_<area>.py` | pytest |
| `test_<feature>.py` in `integration/` | Playwright |
| Factories in `tests/base.py` | `MapFactory`, `DataLayerFactory` |

### What uMap does **not** do

- No `src/` monorepo layout
- No React/Vue frontend
- No Django REST Framework
- No per-feature Python packages — features are cross-cutting files

---

## Call graph cheat sheet

### Opening a map

```
urls.py → MapView
  → models.Map + datalayers.metadata()
  → map_detail.html → map_init.html
  → App.init()
  → DataLayer constructor (metadata only)
  → DataLayer.fetchData() → GET datalayer_view
  → LeafletProxy.addFeature(...)
```

### Saving edits

```
User Ctrl+S
  → App.saveAll()
  → Journal.save()
  → DataLayer.save() / Map.save() (client methods)
  → POST datalayer_update / map_update
  → forms → models → FSDataStorage new .geojson version
```

### Importing remote Overpass layer

```
Importer dialog → importers/overpass.js
  → sets remoteData.url + format=osm
  → DataLayer.fetchRemoteData()
  → optional ajax_proxy
  → Formatter.parse → fromGeoJSON
```

---

## Where first contributions usually land

| If you are working on… | Start in… | Also check… |
|---|---|---|
| Map control / toolbar | `ui/controls.js`, `ui/bar.js` | `schema.js` if new setting |
| Edit panel / map settings | `form/builder.js`, `schema.js` | `MapUpdate` view |
| Drawing markers/lines | `umap.controls.js`, `features.js` | Playwright `test_draw_*` |
| Layer save/load bugs | `data/layer.js`, `views.py` DataLayer* | `storage/fs.py`, `test_datalayer_views.py` |
| Import format | `formatter.js` | `test_import.py` |
| New import helper | `importers/yours.js`, `importer.js` switch | `docs/config/importers.md` |
| Permissions UX | `permissions.js` | `decorators.py`, `models.can_edit` |
| Popup/template variables | `features.js`, `templates.js` | FAQ in docs-users |
| Remote data / proxy | `data/layer.js`, `AjaxProxy` | `test_remote_data.py` |
| Choropleth/cluster | `data/types.js`, `rendering/layers/` | `test_choropleth.py`, `test_cluster.py` |
| Backend search/homepage | `views.py`, `managers.py` | `test_map_views.py` |

### Safe to defer

| Area | Why |
|---|---|
| `sync/` + Redis | Optional feature, complex distributed state |
| `openlayers.js` | Migration in flux |
| `charts/`, `docker/` | Ops, not product logic |
| Most `management/commands/` | Admin/maintenance tooling |

---

## Phase 4 summary

### MUST REMEMBER

1. **`app.js` is the client hub** — not `umap.js` (outdated docs).
2. **`schema.js` registers settings** — new options start here.
3. **`views.py` + `data/layer.js`** are the save/load contract.
4. **`umap.controls.js` is the draw-tool bridge** — legacy global script.
5. **Tests mirror features** — `integration/test_<feature>.py` is your safety net.

### USEFUL LATER

- `journal/` internals
- `management/commands/`
- Helm/Docker paths

### IGNORE FOR NOW

- Full `vendors/` tree
- Translation pipeline (`tx pull`, `compilemessages`) until doing i18n
- `charts/umap` unless deploying

---

## What remains uncertain

| Topic | Status |
|---|---|
| Full CI test workflow location | **PARTIALLY KNOWN** — `Makefile` defines tests; `.github/workflows/` here is mostly docs/helm — upstream may run tests elsewhere |
| Whether all instances enable same `UMAP_IMPORTERS` | **UNKNOWN** — deployment-specific |
| Complete list of query-string embed params | **INFERENCE** — scattered in `app.setPropertiesFromQueryString()` — Phase 5 |

---

## What we investigate next — Phase 5: Vertical Runtime Walkthroughs

We pick 2–4 real user actions and trace them end-to-end with exact functions:

1. **Open an existing map** (likely first — load path)
2. **Draw a marker and save**
3. **Import or remote layer fetch**
4. **Change permissions or embed**

Each flow gets a Mermaid diagram and step-by-step narration.

**Default recommendation:** Start with **open map** + **save marker** unless you prefer frontend-heavy or backend-heavy ordering.

---

## Pause here

You should now know **where to look** without a full tree listing.

**Questions before Phase 5:**

- Frontend-weighted or backend-weighted walkthrough order?
- Any module from this tour that still feels opaque (`journal`, `schema`, `datalayer_save` naming)?
- Want the first walkthrough to include **network payloads** (FormData shape)?

When ready, say **"continue to Phase 5"** or name which flow to trace first.

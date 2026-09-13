# uMap Onboarding — Phase 1: What is uMap?

> **Status:** Phase 1 of 13 · Read-only investigation · Evidence from checked-out repository  
> **Audience:** Experienced web engineer (React/TS/Node) learning uMap for meaningful OSS contribution

---

## Before we look at files

Here is the mental model you need to carry into the repository.

**uMap is a web application for authoring and publishing thematic maps.** A user starts from an OpenStreetMap-based basemap, adds their own geographic content (markers, lines, polygons, imported datasets), organizes that content into layers, styles it, controls who can see or edit it, and then shares or embeds the result on a website.

It is **not** a general-purpose GIS desktop tool, **not** an OpenStreetMap editor (like iD or JOSM), and **not** merely a map viewer. The product sits in a narrower lane: **fast, browser-based map storytelling and lightweight data visualization**, aimed especially at people who want OSM as a backdrop but need a custom overlay they can publish in minutes.

Think of it like this if you come from React/Firebase:

| Familiar concept | uMap analogue |
|---|---|
| A Firebase-hosted SPA project | A **Map** — the top-level document users create |
| Firestore collections / subcollections | **Data layers** — grouped geographic features inside a map |
| Documents with JSON fields | **Features** — points, lines, polygons with properties (name, description, custom fields) |
| Security rules (read/write) | **Share status** and **edit status** on maps and layers |
| Embedding a widget in another site | **iframe embed** with URL parameters controlling controls |

That comparison is intentionally loose — uMap is server-rendered Django plus a Leaflet client, not a Firebase app — but it captures the *product shape*: **one authored artifact, nested data groups, permissions, publish/embed**.

---

## What uMap is

**FACT:** The project README describes uMap as: *"lets you create maps with OpenStreetMap layers within minutes and embed them in your site."* It is built on **Django** (server) and **Leaflet** (client map library).

**FACT:** `pyproject.toml` keywords include `django`, `leaflet`, `geodjango`, `openstreetmap`, `map`.

**FACT:** The developer overview (`docs/dev/overview.md`) states uMap is **a server and a client**. Most geographic data is stored as **GeoJSON files** on the server; users, permissions, and related metadata live in a **database** (PostgreSQL with PostGIS for some geo features). Most map computation happens on the **frontend** with Leaflet.

**INFERENCE:** uMap is designed to be **self-hostable** (install docs, Docker, Helm, etc. exist) while also running as a **public hosted service** — the best-known instance is [umap.openstreetmap.fr](https://umap.openstreetmap.fr/), referenced throughout user tutorials.

---

## What problem it solves

OpenStreetMap gives the world a shared, editable basemap. But many people — associations, journalists, municipalities, event organizers, activists — need something more specific:

- "Show *our* festival venues on top of OSM"
- "Draw a bike route and share it"
- "Import library locations and let visitors toggle layers"
- "Embed an interactive map on our WordPress site"

**FACT:** The README motivation line is explicit: *"Because we think that the more OSM will be used, the more OSM will be improved."*

uMap lowers the barrier between **having geodata** and **publishing an interactive map**. Users do not need GIS software, tile servers, or custom Leaflet code. They get drawing tools, import assistants, styling, permissions, and embed codes in one product.

---

## Who uses it

**FACT (from user documentation and tutorials):** Documented user segments include:

- **General public and associations** — using the OSM community instance (`umap.openstreetmap.fr`)
- **Public-sector agents (France)** — a dedicated instance (`umap.incubateur.anct.gouv.fr`) with ProConnect authentication
- **Anyone embedding maps** — HTML iframe, WordPress, etc. (tutorial 7)
- **Map authors working with OpenStreetMap data** — Overpass queries, GeoDataMine imports, administrative boundaries (tutorials 6, 11)

**INFERENCE:** Typical users are **non-programmers** who need publishable maps, plus **technical users** who want quick thematic mapping without building a custom app. As a contributor, you are on the engineering side of a product primarily shaped by cartographer/journalist/association needs.

**UNKNOWN (from this repo alone):** Exact usage statistics, geographic distribution, or commercial vs. community breakdown. The project is community-funded (Liberapay, OpenCollective, GitHub Sponsors badges in README).

---

## Most important user journeys

These are the flows worth holding in your head. We will trace them through code in Phase 5.

### 1. Browse / consume a map (viewer)

**FACT:** A viewer opens a shared URL, pans and zooms, toggles layers on/off, clicks features to see popups (text, images, links), and may open an "About" or "Browse data" side panel.

This is the **read-only** experience — no account required for public maps.

### 2. Create and edit a map (author)

**FACT:** Authors enter **edit mode** (`Ctrl+E` per FAQ). In edit mode they can:

- Name the map and write a description/legend
- Draw **markers**, **lines**, **polygons**, and routes
- Configure basemap/tile layers and map settings
- **Save** (`Ctrl+S`)
- **Undo** back to last save (`Ctrl+Z`)

**FACT:** Maps can be created **without an account**. Anonymous maps receive a **secret edit link**; only people with that link can modify them (tutorial 2, French).

### 3. Organize content into layers

**FACT:** Features live in **data layers** (calques). Layers can be shown/hidden, reordered, described in the legend, and targeted when drawing or importing. One map can present different layer subsets in different embed "views" (tutorial 6).

**MUST UNDERSTAND NOW:** **Map → DataLayer(s) → Feature(s)** is the core content hierarchy.

### 4. Import external data

**FACT:** uMap provides import assistants for things like French administrative boundaries, GeoDataMine POI datasets, file upload (GeoJSON, etc.), and **Overpass** queries against OpenStreetMap (tutorials 6, 11).

A critical distinction from tutorial 11:

- **Copied into layer** — data is stored in uMap (in the layer's GeoJSON file)
- **Remote data** — data is **not stored on the uMap server**; the client fetches it live (e.g. Overpass API, remote GeoJSON URL)

### 5. Publish, share, embed

**FACT:** Authors control **access status** (draft, public, link-only, editors-only) and **edit status** (owner, collaborators, everyone, secret link). They can export/share via iframe embed with many URL toggles (scale control, layer control, edit mode, etc.) — tutorial 7.

**FACT:** Unauthorized viewers get **403 Forbidden** (FAQ).

### 6. Account-based catalog (optional)

**FACT:** Users can create accounts to maintain a personal catalog of maps instead of relying on secret edit links (tutorial 3, referenced from tutorial 2).

**USEFUL LATER:** Teams, collaborators, templates — we will see these in the data model when we reach architecture.

---

## What kind of data users manipulate

### The Map (top-level document)

**FACT:** In `umap/models.py`, a `Map` has: name, slug, center point, zoom, owner, editors, team, share/edit statuses, tags, licence, and a `settings` JSON field for map-level configuration (description, tile layers, UI options, etc.).

Think: **the canvas + metadata + permissions envelope**.

### Data layers

**FACT:** A `DataLayer` belongs to a `Map`, has a UUID primary key, name, description, rank (ordering), settings JSON, permissions, and a **`geojson` file field** — the layer's features are primarily persisted as a GeoJSON file on disk (or object storage).

Think: **a named group of features**, like a layer in Photoshop or a collection in Firestore.

### Features (inside GeoJSON)

**FACT:** Features are standard GeoJSON geometry types — **Point** (markers), **LineString** (lines/routes), **Polygon** (areas) — with properties: name, description, custom columns, styling hints.

**FACT:** The FAQ documents **conditional styling rules** on feature properties (`population>10000` → color red), **template variables** (`{lat}`, `{name}`, `{osm_id}`, etc.), and rich text in descriptions (markdown-like syntax).

### Remote vs. stored data

**FACT:** Not all visible data lives in uMap storage. Layers can reference **remote URLs** or **dynamic Overpass queries** that the browser fetches at runtime (tutorial 11).

**MUST UNDERSTAND NOW:** uMap is both a **storage system for authored GeoJSON** and a **client for live geodata sources**.

### Basemap / tile layers

**FACT:** Maps use **tile layers** (OSM and alternatives) as the background. The default tile layer is wired into map preview settings in the `Map` model's `preview_settings` property.

Think: **the wallpaper behind your stickers** — separate from your authored features.

---

## What makes uMap different from "just viewing a map"

| Just viewing OSM (openstreetmap.org) | uMap |
|---|---|
| One global map of the planet | A **custom map document** with its own URL |
| OSM data only | OSM **basemap** + **your overlays** |
| Edit OSM itself | Edit **your layers** (not replacing JOSM/iD) |
| No embeddable themed product | **iframe embed** with configurable controls |
| No per-project permissions | **Share/edit statuses** per map and layer |
| No styling pipeline for arbitrary overlays | **Per-feature and per-layer styling**, conditional rules, templates |

**INFERENCE:** uMap competes in the space of "quick web map publishing" (alongside tools like Google My Maps, Carto, etc.) but is **open source**, **OSM-native**, and **self-hostable** — aligned with the OSM community ethos.

---

## Where OpenStreetMap fits

OSM plays **three distinct roles** in uMap. Keeping them separate will save you confusion later.

### 1. Basemap tiles

**FACT:** README and tutorials describe maps "with OpenStreetMap layers" — OSM-derived **raster tiles** are the default visual background.

**Analogy:** Like using a Google Maps tile layer under your custom Firebase geo points — you see OSM streets and labels, but that's not your data.

### 2. Data source via Overpass

**FACT:** Tutorial 11 walks through writing **Overpass QL** queries (via Overpass Turbo), adapting output format (`[out:xml]` because uMap understands OSM XML, not Overpass JSON), and attaching the query as a **remote data layer**.

**Analogy:** Like pointing your React app at a live REST API instead of copying rows into Firestore — the map updates from OSM's live database.

### 3. Ecosystem alignment, not editing

**FACT:** uMap encourages OSM *usage* (README tagline). Import assistants pull OSM-sourced datasets (GeoDataMine, etc.).

**FACT:** uMap is **not** an OSM editor. You draw/import *on top of* OSM; editing OSM itself is out of scope.

**USEFUL LATER:** `osm_type` and `osm_id` feature variables (FAQ) appear when data originated from OSM.

---

## Concepts you need before the code will make sense

### MUST UNDERSTAND NOW

1. **Map** — one publishable map project (URL, settings, permissions)
2. **Data layer** — grouped features within a map; persisted as GeoJSON file + DB metadata
3. **Feature** — a point, line, or polygon with properties; GeoJSON geometry + attributes
4. **Edit mode vs. preview/browse mode** — authoring UI vs. visitor UI
5. **GeoJSON** — the interchange format for stored layer data (developer overview)
6. **Remote data layer** — client-fetched data not stored in uMap's GeoJSON files
7. **Share status / edit status** — authorization model for viewing and editing
8. **Leaflet** — the browser map rendering library uMap builds on (not React — **vanilla JavaScript** per dev overview)

### USEFUL LATER (Phase 2 will teach these properly)

- Coordinate systems, projections, bounding boxes (`{bbox}` variables in FAQ)
- Tile pyramids and zoom levels
- PostGIS / Django GIS fields (center point on `Map`)
- Versioned GeoJSON files (storage layer hints in `DataLayer.save`)
- Django models, views, URL routing
- Playwright integration tests, pytest, Mocha JS tests

### Explicitly defer

- Deployment topologies (Docker, Helm, Dokku) — not needed to understand the product
- Translation workflow (Transifex) — contribution path, not product core
- Full importer plugin architecture — until we trace import flows

---

## Compact mental model (keep this in your head)

```
┌─────────────────────────────────────────────────────────────┐
│                         uMap Product                        │
├─────────────────────────────────────────────────────────────┤
│  Author creates a MAP                                       │
│    ├── settings: center, zoom, basemap, legend, UI opts    │
│    ├── permissions: who can see / edit                      │
│    └── DATA LAYERS (ordered)                                │
│          ├── Layer A: GeoJSON file on server (stored)       │
│          │     └── Features: points, lines, polygons        │
│          ├── Layer B: remote URL or Overpass (live fetch)   │
│          └── Layer C: imported admin boundaries, etc.     │
├─────────────────────────────────────────────────────────────┤
│  Viewer opens URL → Leaflet renders basemap + layers        │
│  Author embeds iframe → same map, constrained controls      │
└─────────────────────────────────────────────────────────────┘

         OpenStreetMap ──► basemap tiles
                         └──► optional live data (Overpass)
                         └──► community context / mission
```

**One sentence version:** uMap is a **collaborative map publishing tool** where each **Map** is a configured Leaflet experience over OSM tiles, containing **layers** of **GeoJSON features** (or remote data), with **permissions** and **embed/share** as first-class concerns.

---

## Documentation caveat

**FACT:** Most English user tutorials in `docs-users/tutorials/` are stubs pointing to French translations. The French tutorials in `docs-users/fr/tutorials/` contain the substantive user-facing content. Developer docs in `docs/` are in English.

When we teach from user journeys, we will cite French tutorials as **authoritative product documentation in this repository**, not infer behavior from generic map apps.

---

## Phase 1 summary — what to remember

1. **uMap authors thematic maps** on OSM basemaps and publishes/embeds them — it is not an OSM editor.
2. The content hierarchy is **Map → DataLayer → Feature (GeoJSON)**.
3. Data can be **stored** (GeoJSON files) or **remote** (Overpass, URLs).
4. The stack is **Django server + vanilla JS/Leaflet client**; most geo computation is client-side.
5. Permissions (share/edit status) are a core product feature, including anonymous maps with secret edit links.
6. Your React/TS skills transfer to **reading UI patterns and data flow**, but the map editor is **not a React app**.

---

## What remains uncertain (honest gaps)

| Topic | Status |
|---|---|
| Exact frontend module structure | **UNKNOWN** until Phase 3–5 (dev overview only says "vanilla JavaScript") |
| How much PostGIS is used vs. file storage | **PARTIALLY KNOWN** — overview says "for the most part" frontend; `Map.center` uses `PointField` |
| Whether hosted instances differ feature-wise from self-hosted | **INFERENCE** — likely configuration/auth differences; not fully documented in one place |
| Current default auth providers | **UNKNOWN** — `social-auth-app-django` in dependencies suggests OAuth; needs Phase 3/7 |
| English tutorial coverage timeline | **UNKNOWN** |

---

## What we investigate next — Phase 2: Domain Primer

Before tracing code paths, we will teach **only the geospatial concepts this repository actually uses**:

- GeoJSON structure as uMap stores it
- Points, lines, polygons in the editor
- Layers, styling, conditional rules
- Tiles and basemaps (Leaflet context)
- Bounding boxes and dynamic remote queries
- OSM / Overpass / OSM XML as remote data formats

We will skip generic GIS lectures and tie every concept to a file, tutorial, or model field.

---

## Pause here

Phase 1 is intentionally product-only — no directory tree, no runtime walkthrough yet.

**Questions worth asking before Phase 2:**

- Anything unclear about Map vs. DataLayer vs. Feature?
- Do you want Phase 2 to lean harder on Leaflet concepts (since you know React but may not know Leaflet)?
- Are you planning to contribute primarily to **frontend (JS)**, **backend (Django/Python)**, or **unsure**?

When you are ready, say **"continue to Phase 2"** (or ask questions).

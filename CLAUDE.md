# Busca Ricardo Claro — Search Coordination App

> Single-file HTML dashboard for coordinating a civilian missing persons search in the Faro/Olhão/Moncarapacho corridor (Algarve, Portugal). Live and in active use.

---

## What this is

A real-time search coordination tool for ~40 civilian searchers looking for Ricardo Claro (missing since 13 Mar 2026). It is a **single HTML file** with no build system. It runs by opening in a browser or deploying to Netlify via drag-and-drop.

**This app is in live use during an active search. Never break existing functionality.**

---

## Tech Stack

- **Leaflet.js 1.9.4** — map rendering (CDN)
- **Supabase JS 2.39.7** — real-time GPS location sync (CDN)
- **CartoDB Light** — map tiles
- **No build system** — pure HTML/CSS/JS, single file
- **Deploy** — drag `index.html` onto Netlify

---

## File Structure

```
index.html                  ← rename from search-map.html before deploying
docs/
  superpowers/plans/
    2026-03-21-persistence-sync-mobile-export.md  ← active implementation plan
```

Everything is in `index.html`. There are two `<script>` blocks:
- **Main script** (~600 lines): grid generation, map, zone/team/log state, people table
- **Supabase script** (~300 lines): GPS tracking, mode selection, live markers, people/teams/zone DB sync

---

## Running locally

No install needed. Open `index.html` in any browser:

```bash
# Windows
start index.html

# Or just double-click index.html in Explorer
```

For Supabase features (live GPS), the credentials in the file must be replaced first (see Configuration below).

---

## Deploying

```
1. Rename search-map.html → index.html
2. Go to app.netlify.com → Sites
3. Drag index.html onto the deploy zone
4. Done — URL is live immediately
```

---

## Configuration

Two placeholders at the bottom of `index.html` (Supabase script block):

```javascript
const SUPABASE_URL = 'REPLACE_WITH_YOUR_SUPABASE_URL';
const SUPABASE_KEY = 'REPLACE_WITH_YOUR_SUPABASE_ANON_KEY';
```

**The app works fully without Supabase** — every Supabase call is wrapped in `if (!sb) return`. Without credentials, GPS tracking is disabled but all zone/team management works locally.

### Supabase tables (all created and active)

```sql
-- GPS tracking
CREATE TABLE locations (
  name text PRIMARY KEY, lat float8, lng float8,
  accuracy int4, updated_at timestamptz
);

-- Zone state persistence
CREATE TABLE zone_states (
  id int PRIMARY KEY,
  status text NOT NULL DEFAULT 'pending',
  team text NOT NULL DEFAULT '',
  searches jsonb NOT NULL DEFAULT '[]',
  updated_at timestamptz DEFAULT now()
);

-- People list
CREATE TABLE people (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  phone text,
  team text,
  zone integer,
  updated_at timestamptz DEFAULT now()
);

-- Teams (currently unused as a standalone entity — team names live as strings in people.team)
CREATE TABLE teams (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  lead text,
  size integer,
  zone text,
  "lastCheckin" text,
  updated_at timestamptz DEFAULT now()
);
```

All tables have RLS enabled with a `public_access` policy (`FOR ALL USING (true) WITH CHECK (true)`).

---

## Key Functions

| Function | Where | What it does |
|----------|-------|-------------|
| `makeGrid()` | main script | Generates ~425 uniform zone cells (0.014°lat × 0.018°lng each) |
| `renderLayers()` | main script | Draws all polygons on the Leaflet map |
| `renderAll(redrawMap)` | main script | Updates sidebar; only redraws map if `redrawMap=true` |
| `setS(id, status, team, dt)` | main script | Sets one zone's status + logs history entry |
| `bulkSet(status)` | main script | Sets multiple zones at once from comma-separated input |
| `now()` / `nowFull()` | main script top | Time helpers — **must stay at top of script**, used during init |
| `renderPeopleTable()` | main script | Renders Pessoas tab table; also syncs team datalist + filter |
| `ptAddPerson()` | main script | Add person from inline form row; saves to localStorage + Supabase |
| `ptEditCell(td, pid, field)` | main script | Inline cell edit (name/phone/team); saves to localStorage + Supabase |
| `ptSetZone(pid, val)` | main script | Zone select dropdown handler; saves to localStorage + Supabase |
| `deletePerson(id)` | main script | Remove person; deletes from localStorage + Supabase |
| `loadZoneStates()` | supabase script | Load all zone states from Supabase on startup |
| `saveZoneState(z)` | supabase script | Persist one zone to Supabase |
| `subscribeToZoneChanges()` | supabase script | Realtime sync between devices |
| `loadPeople()` | supabase script | Load people from Supabase on coordinator startup; overwrites localStorage |
| `savePersonDb(p)` | supabase script | Upsert one person to Supabase `people` table |
| `deletePersonDb(id)` | supabase script | Delete person from Supabase by id |
| `loadTeams()` | supabase script | Load teams from Supabase on coordinator startup |
| `saveTeamDb(t)` | supabase script | Upsert one team to Supabase `teams` table |
| `startCoordinator()` | supabase script | Coordinator mode: polls GPS, loads zone/people/teams state |
| `startSearcher()` | supabase script | Searcher mode: sends GPS to Supabase every 30s |

---

## Key Data Structures

```javascript
// Zone
{ id: Number, p: 'high'|'mid'|'low', name: String, focus: String,
  t: '3–4h', status: 'pending'|'active'|'done'|'flagged',
  team: String,
  searches: [{ status, team, datetime, note }],  // newest first (unshift)
  poly: [[lat,lng], [lat,lng], [lat,lng], [lat,lng]] }

// Person (Pessoas tab)
{ id: Number (Date.now()), name: String, phone: String|null,
  team: String|null, zone: Number|null }  // zone is zone.id

// Team (teams[] array — team names are strings in person.team; teams[] currently not heavily used)
{ id: Number, name: String, lead: String, size: Number,
  zone: String, lastCheckin: String }

// Log entry
{ time: String, text: String, type: 'normal'|'ok'|'flag', team: String }

// Supabase locations row
{ name: String (PK), lat: Float, lng: Float, accuracy: Int, updated_at: Timestamptz }
```

---

## Grid Constants

```javascript
const DL = 0.014;   // ~1.55km per cell (latitude)
const DG = 0.018;   // ~1.45km per cell (longitude)
const LAT0 = 36.974, LNG0 = -8.100;  // SW corner
const LAT1 = 37.212, LNG1 = -7.656;  // NE corner
// Result: ~425 uniform cells
```

The grid is uniform — all cells are identical size. Do not reintroduce a multi-tier system (it caused overlapping polygons and was removed).

---

## Critical Gotchas

**`now()` and `nowFull()` must be defined first.**
They are called during zone initialisation. If moved below the grid code, the app crashes silently on load.

**`redrawMap` flag in `renderAll()`.**
Always pass `false` when only updating the sidebar (e.g. filter changes). Pass `true` only when zone status changes. Passing `true` on every call causes visible flicker and performance issues.

**Single `zoomend` listener.**
The listener is registered once, outside `renderLayers()`. Do not move it inside — it was previously registered once per zone (~1500 listeners) which caused memory leaks and sluggish zoom.

**`searches[]` is newest-first.**
New entries are added with `unshift()`. Display code reads `searches[0]` as the latest. Do not change to push/pop.

**Supabase `if (!sb) return` pattern.**
Every function that touches Supabase must start with this check. The app must remain fully functional with `sb = null`.

**People persistence: localStorage + Supabase.**
`savePeople()` writes to localStorage (instant, offline). `savePersonDb(p)` upserts to Supabase (async, may fail silently). Both are called together. On coordinator startup, `loadPeople()` overwrites localStorage with Supabase data if any exists — this is the source of truth.

**Team names are strings, not objects.**
`teams[]` array exists but team identity in practice is the `person.team` string. The datalist in the Pessoas tab combines both `teams[].name` and all distinct `people[].team` values so autocomplete always shows all known teams.

**Zone IDs are 1-indexed integers.**
`zones[i].id === i + 1`. The `makeGrid()` function reassigns IDs after generation. Bulk input (comma-separated) uses these IDs directly.

---

## What's Implemented vs. Pending

| Feature | Status |
|---------|--------|
| Grid map (425 zones) | Done |
| Zone status management | Done |
| Bulk zone update | Done |
| Zone history logging | Done |
| Activity log | Done |
| Live GPS tracking (Supabase) | Done (requires credentials) |
| Coordinator / Searcher mode select | Done |
| Zone state persistence (Supabase) | Done |
| Realtime zone sync between devices | Done |
| uMap overlay (Pesquisado + Layer 2) | Done |
| People table (inline edit, zone assign) | Done |
| People persistence (Supabase) | Done |
| **CSV export** | **Pending** |
| **Mobile responsive layout** | **Pending** |

---

## uMap Layers

O mapa uMap `1379403` tem 2 layers, ambas carregadas em paralelo pela app:
- `e0c5c3aa-50ae-41bd-b024-a5102ef8b10f` — "Pesquisado" (rotas e áreas pesquisadas)
- `8c7289c8-3bfe-4374-9f23-f1b36971b2e7` — "Layer 2" (zonas com marcadores e polígonos)

URL pattern: `https://umap.openstreetmap.fr/en/datalayer/1379403/<UUID>/`
Implementado em `UMAP_URLS` (array) + `fetchOneLayer()` + `Promise.all` em `loadUmapLayer()`.

---

## Search Area Reference

| Location | Coordinates | Notes |
|----------|-------------|-------|
| Last sighting | 37.016, -7.935 | Faro, 13 Mar ~21:00 |
| Car found | 37.026, -7.841 | Olhão, 19 Mar |
| PJ search focus | 37.070, -7.787 | Moncarapacho |
| Ricardo's workplace | ~37.024, -8.136 | Well restaurant, Vale do Lobo |

---

## Do Not Touch

- The `redrawMap` flag logic in `renderAll()` — it prevents infinite re-render loops
- The `zoomend` listener location (outside `renderLayers()`)
- The `now()` / `nowFull()` position at the very top of the main script
- Supabase RLS policies — they are intentionally open (public search coordination, no sensitive data)
- Anything that would break offline/no-Supabase mode

---

## Sensitivity

- Do not add suspect names or case details to UI text or code comments
- Do not log GPS coordinates in browser console in production
- Media coordination goes through the family (Sandra Primitivo, Ricardo's sister)
- The PJ investigation is separate from civilian search — do not surface case details in the app

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
- **Main script** (~500 lines): grid generation, map, zone/team/log state
- **Supabase script** (~200 lines): GPS tracking, mode selection, live markers

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

### Supabase tables required

```sql
-- GPS tracking (already exists if GPS was ever set up)
CREATE TABLE locations (
  name text PRIMARY KEY, lat float8, lng float8,
  accuracy int4, updated_at timestamptz
);

-- Zone state persistence (to be added — see implementation plan)
CREATE TABLE zone_states (
  id int PRIMARY KEY,
  status text NOT NULL DEFAULT 'pending',
  team text NOT NULL DEFAULT '',
  searches jsonb NOT NULL DEFAULT '[]',
  updated_at timestamptz DEFAULT now()
);
```

Both tables need RLS enabled with a public_access policy (`FOR ALL USING (true) WITH CHECK (true)`).

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
| `loadZoneStates()` | supabase script | Load all zone states from Supabase on startup (to be implemented) |
| `saveZoneState(z)` | supabase script | Persist one zone to Supabase (to be implemented) |
| `subscribeToZoneChanges()` | supabase script | Realtime sync between devices (to be implemented) |
| `startCoordinator()` | supabase script | Coordinator mode: polls GPS, loads zone states |
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

// Team
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
| Team management + check-in | Done |
| Activity log | Done |
| Live GPS tracking (Supabase) | Done (requires credentials) |
| Coordinator / Searcher mode select | Done |
| **Zone state persistence** | **Pending — Task 2-3 in plan** |
| **Multi-device realtime sync** | **Pending — Task 4 in plan** |
| **CSV export** | **Pending — Task 5 in plan** |
| **Mobile responsive layout** | **Pending — Task 6 in plan** |

Active plan: `docs/superpowers/plans/2026-03-21-persistence-sync-mobile-export.md`

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

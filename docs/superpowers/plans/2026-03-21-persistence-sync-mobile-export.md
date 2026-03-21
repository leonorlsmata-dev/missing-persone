# Search App: Persistence, Sync, Export & Mobile Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the coordinator app survive page reloads, sync across multiple devices in real-time, export data for PJ handover, and work properly on mobile phones.

**Architecture:** All changes are within the single `index.html` file. Supabase (already in stack for GPS) is extended with a `zone_states` table. JS functions `setS()` and `bulkSet()` gain async persistence calls. A Supabase Realtime channel replaces the need for manual refresh between devices. Export and mobile layout are pure HTML/CSS/JS with no external dependencies.

**Tech Stack:** Leaflet.js 1.9.4, Supabase JS 2.39.7, vanilla HTML/CSS/JS, no build system, deploys by drag-and-drop to Netlify.

> **Scope note:** Tasks 1-3 (Supabase sync) and Tasks 4-5 (UI) are independent. If short on time, stop after Task 3 — it is the highest-value block. Tasks 4-5 can be done in a separate session.

---

## Context: What is broken today

| Problem | Impact |
|---------|--------|
| Zone statuses reset on page reload | All search progress lost if coordinator closes tab |
| No multi-device sync | Second coordinator sees stale data; changes collide |
| No export | Cannot hand over zone log to PJ |
| 300px sidebar on mobile | Coordinator on phone cannot use the app comfortably |

---

## Files

All changes are in one file: `index.html` (currently named `search-map.html` — rename it).

| Section | What changes |
|---------|-------------|
| `<style>` block | Add mobile responsive rules (~40 lines) |
| Supabase config block (bottom `<script>`) | Add `loadZoneStates()`, `saveZoneState()`, `subscribeToZoneChanges()` |
| Main `<script>` | Modify `setS()` and `bulkSet()` to call save; add `exportCSV()` |
| Header HTML | Add Export button; add mobile sidebar toggle button |
| Supabase SQL (run once in dashboard) | Add `zone_states` table |

---

## Task 1: Supabase — Create zone_states table

**Files:** None (run SQL in Supabase dashboard)

- [ ] **Step 1: Run this SQL in Supabase SQL Editor**

```sql
CREATE TABLE zone_states (
  id int PRIMARY KEY,
  status text NOT NULL DEFAULT 'pending',
  team text NOT NULL DEFAULT '',
  searches jsonb NOT NULL DEFAULT '[]',
  updated_at timestamptz DEFAULT now()
);
ALTER TABLE zone_states ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_access" ON zone_states
  FOR ALL USING (true) WITH CHECK (true);
```

- [ ] **Step 2: Verify table exists**

In Supabase dashboard: Table Editor → should show `zone_states` with 0 rows.

---

## Task 2: Load zone states from Supabase on startup

**Files:** Modify `index.html` — bottom `<script>` block (Supabase section), near `startCoordinator()`.

- [ ] **Step 1: Add `loadZoneStates()` function**

Place this after the `colorFor()` helper, before `selectMode()`:

```javascript
async function loadZoneStates() {
  if (!sb) return;
  try {
    const { data, error } = await sb.from('zone_states').select('*');
    if (error) throw error;
    (data || []).forEach(row => {
      const z = zones.find(x => x.id === row.id);
      if (!z) return;
      z.status = row.status;
      z.team = row.team || '';
      z.searches = row.searches || [];
    });
    renderAll(true);
    addLog(`Estado das zonas carregado (${data.length} registos)`, 'ok');
  } catch(e) {
    console.warn('loadZoneStates error:', e);
  }
}
```

- [ ] **Step 2: Call `loadZoneStates()` inside `startCoordinator()`**

Find the existing `startCoordinator()` function. Add one line after `fetchLocations()`:

```javascript
function startCoordinator() {
  document.getElementById('tracker-panel').style.display = 'block';
  // ... existing button creation code ...
  fetchLocations();
  loadZoneStates();   // <-- ADD THIS LINE
  trackerInterval = setInterval(fetchLocations, 15000);
}
```

- [ ] **Step 3: Manual verification**

1. Open `index.html` in browser (Supabase must be configured)
2. Select "Coordenador" mode
3. Open browser DevTools → Console
4. Should see: `Estado das zonas carregado (0 registos)` in the activity log
5. No errors in console

---

## Task 3: Save zone status changes to Supabase

**Files:** Modify `index.html` — main `<script>` block, `setS()` and `bulkSet()` functions.

- [ ] **Step 1: Add `saveZoneState()` helper**

Place this right before `setS()` in the main `<script>`:

```javascript
async function saveZoneState(z) {
  if (!sb) return;
  try {
    await sb.from('zone_states').upsert({
      id: z.id,
      status: z.status,
      team: z.team || '',
      searches: z.searches,
      updated_at: new Date().toISOString()
    }, { onConflict: 'id' });
  } catch(e) {
    console.warn('saveZoneState error:', e.message);
  }
}
```

- [ ] **Step 2: Call `saveZoneState()` at the end of `setS()`**

Find `setS()`:

```javascript
function setS(id, status, teamName, datetime) {
  const z = zones.find(x => x.id === id); if (!z) return;
  // ... existing code ...
  renderAll(true);
  saveZoneState(z);   // <-- ADD THIS LINE (after renderAll)
}
```

- [ ] **Step 3: Call `saveZoneState()` for each zone in `bulkSet()`**

Find the `ids.forEach` loop inside `bulkSet()` and add the save call:

```javascript
ids.forEach(id => {
  const z = zones.find(x => x.id === id);
  if (z) {
    if (!z.searches) z.searches = [];
    z.searches.unshift({ status, team: teamName, datetime: dt, note: '' });
    if (z.status !== status) { z.status = status; count++; }
    saveZoneState(z);   // <-- ADD THIS LINE
  }
});
```

- [ ] **Step 4: Manual verification**

1. Open `index.html` in browser, select Coordenador
2. Click any zone → mark as "Concluída"
3. Open Supabase dashboard → Table Editor → `zone_states`
4. Should see 1 row with the zone's ID, `status: done`
5. Reload the page, select Coordenador again
6. The zone should load as green (Concluída) — progress survived the reload

- [ ] **Step 5: Commit**

```bash
cd "c:\Users\sofia\Desktop\Missing-persone"
git add index.html
git commit -m "feat: persist zone states to Supabase — survives page reload"
```

---

## Task 4: Real-time sync between devices

**Files:** Modify `index.html` — bottom `<script>` block, `startCoordinator()`.

This makes changes from one device/tab immediately visible on all others.

- [ ] **Step 1: Add `subscribeToZoneChanges()` function**

Place this after `loadZoneStates()`:

```javascript
function subscribeToZoneChanges() {
  if (!sb) return;
  sb.channel('zone_states_changes')
    .on('postgres_changes',
      { event: '*', schema: 'public', table: 'zone_states' },
      payload => {
        const row = payload.new;
        if (!row) return;
        const z = zones.find(x => x.id === row.id);
        if (!z) return;
        // Only update if the change came from a different device
        // (our own saves will also trigger this — it's safe, just re-applies same state)
        z.status = row.status;
        z.team = row.team || '';
        z.searches = row.searches || [];
        renderAll(true);
      }
    )
    .subscribe();
}
```

- [ ] **Step 2: Call `subscribeToZoneChanges()` inside `startCoordinator()`**

```javascript
function startCoordinator() {
  // ... existing code ...
  fetchLocations();
  loadZoneStates();
  subscribeToZoneChanges();   // <-- ADD THIS LINE
  trackerInterval = setInterval(fetchLocations, 15000);
}
```

- [ ] **Step 3: Enable Realtime on the table in Supabase**

In Supabase dashboard:
1. Go to **Database → Replication**
2. Find `zone_states` table
3. Toggle **Realtime** ON for that table

- [ ] **Step 4: Manual verification**

1. Open `index.html` in two browser tabs, both select Coordenador
2. In Tab A: mark Zone 1 as Concluída
3. In Tab B: Zone 1 should turn green within 1-2 seconds (no refresh needed)

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: real-time zone sync across coordinator devices via Supabase"
```

---

## Task 5: CSV export

**Files:** Modify `index.html` — header HTML + main `<script>` block.

- [ ] **Step 1: Add `exportCSV()` function**

Place near the bottom of the main `<script>`, before `renderAll(false)`:

```javascript
function exportCSV() {
  const headers = ['ID','Nome','Prioridade','Estado','Equipa','Ultima_pesquisa','Historico'];
  const rows = zones.map(z => {
    const last = z.searches.length ? z.searches[0] : null;
    const hist = z.searches.map(s =>
      `${s.datetime}|${s.status}|${s.team}`
    ).join(' // ');
    return [
      z.id,
      `"${z.name.replace(/"/g, '""')}"`,
      z.p,
      z.status,
      `"${(z.team||'').replace(/"/g,'""')}"`,
      last ? `"${last.datetime}"` : '',
      `"${hist.replace(/"/g,'""')}"`
    ].join(',');
  });
  const csv = [headers.join(','), ...rows].join('\r\n');
  const blob = new Blob(['\ufeff' + csv], { type: 'text/csv;charset=utf-8;' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `busca-ricardo-${new Date().toISOString().slice(0,10)}.csv`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
  addLog('CSV exportado', 'ok');
}
```

- [ ] **Step 2: Add Export button to header**

Find the `<div class="header-right">` block in the HTML and add one button:

```html
<div class="header-right">
  <button class="hbtn" onclick="exportCSV()">&#8595; Exportar</button>   <!-- ADD THIS -->
  <button class="hbtn" onclick="openModal('checkin')">✓ Check-in</button>
  <button class="hbtn red" onclick="openModal('report')">⚠ Ocorrência</button>
  <button class="hbtn" onclick="openModal('team')">+ Equipa</button>
</div>
```

- [ ] **Step 3: Manual verification**

1. Open `index.html`, mark a few zones as done
2. Click "Exportar" in the header
3. A `.csv` file should download
4. Open in Excel/LibreOffice — should show zone ID, name, status, history columns
5. Special characters (ã, ç, etc.) should display correctly (BOM ensures this)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add CSV export of all zone statuses and search history"
```

---

## Task 6: Mobile responsive sidebar

**Files:** Modify `index.html` — `<style>` block + header HTML.

- [ ] **Step 1: Add mobile toggle button to header HTML**

Add a button inside `<div class="header-left">`, after the pulse dot group:

```html
<button class="hbtn" id="sidebar-toggle" onclick="toggleSidebar()" style="display:none">&#9776; Zonas</button>
```

- [ ] **Step 2: Add mobile CSS to the `<style>` block**

Add at the very end of the `<style>` block, before `</style>`:

```css
@media (max-width: 768px) {
  #sidebar-toggle { display: block !important; }
  .header-title { font-size: 10px; }
  .header-sub { display: none; }
  .hbtn { font-size: 9px; padding: 4px 7px; }

  .sidebar {
    position: absolute;
    left: -300px;
    top: 50px;
    bottom: 0;
    z-index: 900;
    transition: left 0.25s ease;
    box-shadow: 2px 0 16px rgba(0,0,0,0.15);
  }
  .sidebar.open {
    left: 0;
  }
  #map {
    width: 100vw;
  }
}
```

- [ ] **Step 3: Add `toggleSidebar()` JS function**

Add near the other UI helpers (`switchTab`, `openModal`, etc.):

```javascript
function toggleSidebar() {
  const s = document.querySelector('.sidebar');
  s.classList.toggle('open');
  const btn = document.getElementById('sidebar-toggle');
  btn.textContent = s.classList.contains('open') ? '✕ Fechar' : '☰ Zonas';
}
```

- [ ] **Step 4: Auto-close sidebar when a zone is selected on mobile**

In `selZoneClick()`, add at the end:

```javascript
function selZoneClick(id) {
  selZone = id === selZone ? null : id;
  renderAll(true);
  if (selZone) {
    const z = zones.find(x => x.id === selZone);
    if (z && layers[z.id]) map.fitBounds(layers[z.id].getBounds(), { padding: [80, 80] });
    // Close sidebar on mobile after selecting zone
    if (window.innerWidth <= 768) {
      document.querySelector('.sidebar').classList.remove('open');
      document.getElementById('sidebar-toggle').textContent = '☰ Zonas';
    }
  }
}
```

- [ ] **Step 5: Manual verification**

1. Open `index.html` in browser
2. Open DevTools → toggle device toolbar → select iPhone 12 (390px)
3. Should see: map full screen, "☰ Zonas" button in header
4. Tap "☰ Zonas" → sidebar slides in from left
5. Tap a zone → sidebar closes, map flies to that zone
6. Tap "✕ Fechar" → sidebar slides back out

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: mobile responsive sidebar with toggle for phone coordinators"
```

---

## Task 7: Deploy to Netlify

- [ ] **Step 1: Rename file**

```bash
cp "search-map.html" "index.html"
```

- [ ] **Step 2: Drag `index.html` to Netlify**

Go to app.netlify.com → Sites → drag `index.html` onto the deploy zone.

- [ ] **Step 3: Verify on mobile**

Open the Netlify URL on a phone. Test:
- Mode selection screen appears
- Coordinator mode: zones load with saved states
- Mark a zone done → reload → stays done
- Open same URL on second device → change appears within ~2 seconds

---

## Rollback plan

If Supabase changes break anything, the app has a safe fallback built in:

```javascript
let sb = null;
try {
  if (SUPABASE_URL !== 'REPLACE_WITH_YOUR_SUPABASE_URL') {
    sb = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);
  }
} catch(e) { console.warn('Supabase not configured', e); }
```

Every Supabase call is wrapped in `if (!sb) return` — if `sb` is null the app continues working locally. To rollback: clear Supabase credentials → app reverts to in-memory only mode.

---

## Priority order for a time-constrained session

| Order | Task | Time est. | Why |
|-------|------|-----------|-----|
| 1 | Task 1 (SQL table) | 5 min | Required for everything else |
| 2 | Task 2 (load on startup) | 10 min | Progress survives reload |
| 3 | Task 3 (save on change) | 10 min | Progress survives reload |
| 4 | Task 4 (realtime sync) | 10 min | Multi-device |
| 5 | Task 5 (CSV export) | 10 min | PJ handover |
| 6 | Task 6 (mobile layout) | 15 min | Phone usability |
| 7 | Task 7 (deploy) | 5 min | Goes live |

# NutriTrack Improvements — Design Spec
**Date:** 2026-04-21
**Status:** Approved
**Approach:** Option 2 — Additive pass, minimal structural change

---

## Overview

A thorough improvement pass on `NutriTrack/index.html` (single-file, no build tools, localStorage, GitHub Pages). Covers 9 features across UX, data/intelligence, visual polish, and performance. No breaking schema changes. All Spanish UI text preserved.

---

## 1. Schema & Data

**No breaking changes.** Existing structure `{ version, goals, days }` is untouched.

### Additions

**`db.favorites`** — new top-level array of food name strings.
- Added as `db.favorites ?? []` in `loadDB()` and in `importJSON()`.
- No version bump required — gracefully defaults to empty array on existing data.

**Food entry `grams` field** — optional number stored when a preset is added via portion flow.
- `grams: 150` when added via portion sheet; `grams: null` for manual entries.
- Display-only badge; historical entries without this field render without it.

**Export/import:** `importJSON()` gets one added line to carry over `db.favorites`. Existing merge logic (`Object.keys(db.days)`) unchanged.

---

## 2. UX Features

### 2a. Portion Size — Quick-Add Presets

Tapping a preset row no longer immediately adds the food. Instead:

1. A new `#portionOverlay` opens (same `.sheet` / `.handle` / `.overlay` chrome as existing overlays).
2. Sheet shows: food icon + name, a large grams `<input>` pre-filled to `100`, live macro preview row.
3. Live preview recalculates on every `input` event: `val = base * (grams / 100)`, rounded to 1 decimal for all macros, integer for calories.
4. Confirm → calls `confirmPortion()`: pushes entry with computed values + `grams` field, saves, calls `renderProgress(); renderLog()`, closes both overlays.
5. Cancel → closes portion overlay only, returns to add sheet.

The portion overlay sits above `#addOverlay` in z-index. A module-level `let portionFood = null` holds the selected preset while the sheet is open.

### 2b. Edit Log Entries

Each food item row in `renderLog()` gets an edit button (`✎`) in `.f-right`, alongside the existing delete button.

- Tapping opens `#addOverlay` in **edit mode**:
  - Sheet title: `Editar <span>alimento</span>`
  - All fields pre-filled from the stored entry (name, cal, prot, carbs, fat, fiber, meal). If the entry has a `grams` value, it is shown as a read-only note below the name field — editing grams does **not** recalculate macros in edit mode. The user edits the absolute stored values directly.
  - Confirm button label: `Guardar ✓`, calls `saveEdit(id)`
  - `saveEdit(id)` splices the entry in-place at its original index, saves, calls `renderProgress(); renderLog()`, closes overlay
- Cancel exits without changes.
- Module-level `let editingId = null`. `null` = add mode; a food id = edit mode. `openAdd()` always sets `editingId = null` before opening.

### 2c. Favourites

Each row in the quick-add preset list gets a heart toggle button between the macro badges and the `+` button.

- `♡` (muted) when not favourited; `♥` (accent colour, `var(--accent)`) when favourited.
- Tapping calls `toggleFav(foodName)`: adds/removes from `db.favorites`, saves, calls `renderQuick()` only — no full render.
- `renderQuick()` sort order: favourited items first, then the rest (both groups respect the active category filter and search query).
- When any favourites exist in the visible list, a small divider label `⭐ Favoritos` appears above them.
- Favourites persist across sessions via `db.favorites`.

---

## 3. Intelligence Features

### 3a. Streak Counter

**`getStreak()`** — walks `db.days` backwards from yesterday (today excluded — partial day shouldn't break streak). A day counts if `foods.length > 0`. Stops at the first missing or empty day.

**Placement:** Small pill in `.header-row`, between `.date-pill` and the reset button.
- Format: `🔥 N` (flame emoji + count, no label text)
- Hidden (`display:none`) when streak is 0
- Gets an accent border (`border-color: var(--accent)`) when streak ≥ 7
- Uses same `.date-pill` base styling

### 3b. Weekly Summary (History Tab)

**`renderWeeklySummary()`** — called at the top of `renderHistory()`, renders a card above the per-day list.

**Data:** Looks at the 7 most recent calendar days (including today) that have `foods.length > 0`. Computes per-macro averages across those days.

**Card layout:**
- Title: `Resumen semanal`
- Subtitle: `últimos N días con registro` (N = actual days with data, ≤ 7)
- One row per macro (cal, prot, carbs, fat, fiber): label | average value | `/ goal` | percentage bar
- Bars use existing `.fill` colour classes (accent per macro)
- If fewer than 2 days have data: shows `Aún no hay suficiente historial` instead of rows
- Card is omitted entirely if `db.days` has no entries

---

## 4. Visual Polish

### 4a. Food Item Row — Two-Line Layout

Replaces the crowded `.f-macros` inline row with a two-line structure inside `.f-info`:

**Line 1:** flex row — `.f-name` (truncates with ellipsis, `flex:1`) + optional `.f-grams` badge (`150g`, muted pill) + `.f-cal` (accent colour, right-aligned)

**Line 2:** `.f-chips` — flex row of 4 small pill badges: protein (`accent2`), carbs (`accent3`), fat (`accent4`), fiber (`accent5`). Each pill: `font-size:10px`, `padding: 2px 6px`, `border: 1px solid var(--border)`, `background: var(--card2)`, `border-radius: 20px`.

History detail rows (`hist-food-row`) get the same treatment for consistency.

### 4b. Over-Goal Warning State

`renderProgress()` conditionally applies `.over-goal` class when a macro exceeds 100% of its goal.

**CSS rules (added to stylesheet):**
```css
.ring-cal.over-goal,  .ring-prot.over-goal  { stroke: var(--danger); }
.mini-fill.over-goal                         { background: var(--danger) !important; }
.ring-pct.over-goal                          { color: var(--danger); }
```

**"Quedan" → "Excedido":** When over goal, `.prog-rem` text switches from `Quedan <b>X</b> kcal` to `Excedido <b>X</b> kcal` with `color: var(--danger)` on the `<b>` tag.

### 4c. Animation Tightening

`@keyframes fadeUp`: duration `0.2s ease` → `0.15s cubic-bezier(.4,0,.2,1)`, `translateY(5px)` → `translateY(3px)`.

`@keyframes slideUp` (sheet): unchanged at `0.27s` — appropriate for larger element.

Ring/bar transitions: already at `0.5s cubic-bezier(.4,0,.2,1)` — unchanged.

---

## 5. Performance — Targeted Render Calls

`render()` (full) retained for `importJSON()` and `init()` only.

| Callsite | Before | After |
|---|---|---|
| `quickAdd()` / `confirmPortion()` | `render()` | `renderProgress(); renderLog()` |
| `addManual()` / `saveEdit()` | `render()` | `renderProgress(); renderLog()` |
| `delFood()` | `render()` | `renderProgress(); renderLog()` |
| `saveGoal()` | `render()` | `renderProgress(); renderLog(); renderStatsTab()` |
| Favourite toggle | `render()` | `renderQuick()` only |
| `switchTab('stats')` | none | `renderStatsTab()` |
| `switchTab('history')` | none | `renderHistory()` |

`renderWeeklySummary()` is called only from `renderHistory()` — benefits automatically from lazy tab rendering.

---

## 6. Constraints

- Single HTML file, no external dependencies beyond existing Google Fonts import
- No backend, no build tools, no npm
- All UI text remains in Spanish
- `localStorage` key `nutritrack_db` unchanged
- Export/import JSON flow must work end-to-end after changes
- Existing `{ version, goals, days }` schema must load correctly without migration

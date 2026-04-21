# NutriTrack Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 9 improvements (portion sizing, edit entries, favourites, weekly summary, streak counter, two-line food row, over-goal warning, faster animations, targeted renders) to `NutriTrack/index.html` without breaking existing localStorage data or export/import flow.

**Architecture:** All changes are additive to a single HTML file. Tasks are ordered so each is independently verifiable before moving on: schema first, then pure-JS refactor, then visual, then new features. No external dependencies added.

**Tech Stack:** Vanilla HTML/CSS/JS, localStorage, single file, no build tools.

---

### Task 1: Schema additions — db.favorites + grams field

**Files:**
- Modify: `NutriTrack/index.html` (loadDB, importJSON, default return)

- [ ] **Step 1: Add `db.favorites` guard to `loadDB()`**

  Locate the block ending around line 539 (after `p.goals.fiber = p.goals.fiber ?? 25;`).
  Add one line immediately after:

  ```javascript
  p.goals.carbs = p.goals.carbs ?? 250;
  p.goals.fat   = p.goals.fat   ?? 65;
  p.goals.fiber = p.goals.fiber ?? 25;
  p.favorites   = p.favorites   ?? [];   // ← ADD
  return p;
  ```

- [ ] **Step 2: Add `favorites` to the default fresh-DB return value**

  Locate the final `return` in `loadDB()` (around line 559). Change it to:

  ```javascript
  return { version:1, goals:{ cal:2000, prot:150, carbs:250, fat:65, fiber:25 }, days:{}, favorites:[] };
  ```

- [ ] **Step 3: Add `db.favorites` guard to `importJSON()`**

  Locate the block inside the `reader.onload` try (around line 583-586). Add one line:

  ```javascript
  p.goals.carbs = p.goals.carbs ?? 250;
  p.goals.fat   = p.goals.fat   ?? 65;
  p.goals.fiber = p.goals.fiber ?? 25;
  p.favorites   = p.favorites   ?? [];   // ← ADD
  Object.keys(db.days).forEach(k => { if (!p.days[k]) p.days[k] = db.days[k]; });
  db = p; save(); render();
  ```

- [ ] **Step 4: Verify in browser**

  Open `index.html`. Open DevTools → Application → Local Storage.
  - If no existing data: add any food, then check `nutritrack_db` — confirm `favorites: []` is present.
  - Export JSON, clear localStorage (run `localStorage.clear()` in console), import the JSON back — confirm the app loads with all data intact and no JS errors.

- [ ] **Step 5: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: add db.favorites field with backward-compatible guard"
  ```

---

### Task 2: Targeted render calls + lazy tab rendering

**Files:**
- Modify: `NutriTrack/index.html` (quickAdd, addManual, delFood, saveGoal, switchTab)

- [ ] **Step 1: Replace `render()` in `quickAdd()`**

  Find (around line 873):
  ```javascript
  save(); render();
  if (navigator.vibrate) navigator.vibrate(20);
  closeOverlay('addOverlay');
  ```
  Replace with:
  ```javascript
  save(); renderProgress(); renderLog();
  if (navigator.vibrate) navigator.vibrate(20);
  closeOverlay('addOverlay');
  ```

- [ ] **Step 2: Replace `render()` in `addManual()`**

  Find (around line 914):
  ```javascript
  save(); closeOverlay('addOverlay'); render();
  if (navigator.vibrate) navigator.vibrate(20);
  ```
  Replace with:
  ```javascript
  save(); closeOverlay('addOverlay'); renderProgress(); renderLog();
  if (navigator.vibrate) navigator.vibrate(20);
  ```

- [ ] **Step 3: Replace `render()` in `delFood()`**

  Find (around line 880):
  ```javascript
  state.foods = state.foods.filter(f => String(f.id) !== String(id));
  save(); render();
  ```
  Replace with:
  ```javascript
  state.foods = state.foods.filter(f => String(f.id) !== String(id));
  save(); renderProgress(); renderLog();
  ```

- [ ] **Step 4: Replace `render()` in `saveGoal()`**

  Find (around line 932):
  ```javascript
  save(); closeOverlay('goalOverlay'); render();
  ```
  Replace with:
  ```javascript
  save(); closeOverlay('goalOverlay'); renderProgress(); renderLog(); renderStatsTab();
  ```

- [ ] **Step 5: Add lazy renders in `switchTab()`**

  Find `switchTab` (around line 945) and add two lines at the end:

  ```javascript
  function switchTab(tab) {
    document.getElementById('navHome').classList.toggle('active',    tab==='home');
    document.getElementById('navStats').classList.toggle('active',   tab==='stats');
    document.getElementById('navHistory').classList.toggle('active', tab==='history');
    document.getElementById('mainTab').classList.toggle('hide',      tab!=='home');
    document.getElementById('statsTab').classList.toggle('show',     tab==='stats');
    document.getElementById('historyTab').classList.toggle('show',   tab==='history');
    if (tab === 'stats')   renderStatsTab();   // ← ADD
    if (tab === 'history') renderHistory();    // ← ADD
  }
  ```

- [ ] **Step 6: Verify in browser**

  Add a food, delete a food, edit goals — confirm the UI updates correctly after each action. Switch between all three tabs and confirm each shows current data.

- [ ] **Step 7: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "perf: targeted render calls, lazy tab rendering"
  ```

---

### Task 3: Food item row — two-line layout

**Files:**
- Modify: `NutriTrack/index.html` (CSS, renderLog, renderHistDetail)

- [ ] **Step 1: Add new CSS classes**

  Find the `.f-fiber` CSS rule (around line 138). Add these rules immediately after it:

  ```css
  .f-line1 { display: flex; align-items: baseline; gap: 6px; margin-bottom: 4px; }
  .f-line1 .f-name { flex: 1; margin-bottom: 0; }   /* override existing .f-name margin-bottom:3px */
  .f-line1 .f-cal  { font-size: 12px; color: var(--accent); font-weight: 600; flex-shrink: 0; }
  .f-grams { font-size: 10px; color: var(--muted); background: var(--surface); border: 1px solid var(--border); border-radius: 20px; padding: 1px 6px; flex-shrink: 0; }
  .f-chips { display: flex; gap: 5px; flex-wrap: wrap; }
  .f-chip  { font-size: 10px; padding: 2px 6px; border-radius: 20px; border: 1px solid var(--border); background: var(--card2); }
  ```

- [ ] **Step 2: Update `renderLog()` food item HTML**

  Find the food item template inside `renderLog()` (around lines 678-695). Replace the `.f-info` block:

  **Before:**
  ```javascript
  <div class="f-info">
    <div class="f-name">${f.name}</div>
    <div class="f-macros">
      <span class="f-cal">⚡${f.cal}</span>
      <span class="f-prot">💪${f.prot}g</span>
      <span class="f-carbs">🌾${f.carbs||0}g</span>
      <span class="f-fat">🧈${f.fat||0}g</span>
      <span class="f-fiber">🌿${f.fiber||0}g</span>
    </div>
  </div>
  ```

  **After:**
  ```javascript
  <div class="f-info">
    <div class="f-line1">
      <div class="f-name">${f.name}${f.grams ? `<span class="f-grams">${f.grams}g</span>` : ''}</div>
      <span class="f-cal">⚡${f.cal}</span>
    </div>
    <div class="f-chips">
      <span class="f-chip" style="color:var(--accent2)">💪${Math.round(f.prot*10)/10}g</span>
      <span class="f-chip" style="color:var(--accent3)">🌾${Math.round((f.carbs||0)*10)/10}g</span>
      <span class="f-chip" style="color:var(--accent4)">🧈${Math.round((f.fat||0)*10)/10}g</span>
      <span class="f-chip" style="color:var(--accent5)">🌿${Math.round((f.fiber||0)*10)/10}g</span>
    </div>
  </div>
  ```

- [ ] **Step 3: Update `renderHistDetail()` for consistency**

  Find the `.hist-food-row` template inside `renderHistDetail()` (around lines 806-817). Replace the inner content:

  **Before:**
  ```javascript
  <div class="hist-food-row">
    <span class="hist-food-icon">${f.icon}</span>
    <span class="hist-food-name">${f.name}</span>
    <span class="hist-food-macros">
      <span style="color:var(--accent);font-size:11px">⚡${f.cal}</span>
      <span style="color:var(--accent2);font-size:11px">💪${f.prot}g</span>
      <span style="color:var(--accent3);font-size:11px">🌾${f.carbs||0}g</span>
      <span style="color:var(--accent4);font-size:11px">🧈${f.fat||0}g</span>
    </span>
  </div>
  ```

  **After:**
  ```javascript
  <div class="hist-food-row">
    <span class="hist-food-icon">${f.icon}</span>
    <span class="hist-food-name">${f.name}${f.grams ? ` <span style="font-size:10px;color:var(--muted)">${f.grams}g</span>` : ''}</span>
    <span class="hist-food-macros">
      <span style="color:var(--accent);font-size:11px">⚡${f.cal}</span>
      <span class="f-chip" style="color:var(--accent2)">💪${Math.round(f.prot*10)/10}g</span>
      <span class="f-chip" style="color:var(--accent3)">🌾${Math.round((f.carbs||0)*10)/10}g</span>
      <span class="f-chip" style="color:var(--accent4)">🧈${Math.round((f.fat||0)*10)/10}g</span>
    </span>
  </div>
  ```

- [ ] **Step 4: Verify in browser**

  Add 2–3 foods. Confirm each food item shows:
  - Line 1: name (+ grams badge if applicable) + calories right-aligned
  - Line 2: four macro chips (protein, carbs, fat, fiber) in their accent colours
  Switch to History tab and expand a past day — confirm same chip style appears.

- [ ] **Step 5: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: two-line food item row with macro chips"
  ```

---

### Task 4: Animation tightening + over-goal warning

**Files:**
- Modify: `NutriTrack/index.html` (CSS, renderProgress)

- [ ] **Step 1: Tighten `fadeUp` animation**

  Find (around line 129):
  ```css
  @keyframes fadeUp { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
  ```
  Replace with:
  ```css
  @keyframes fadeUp { from { opacity:0; transform:translateY(3px); } to { opacity:1; transform:translateY(0); } }
  ```

  Find `.food-item { ... animation: fadeUp 0.2s ease; }` (around line 128):
  ```css
  animation: fadeUp 0.2s ease;
  ```
  Replace with:
  ```css
  animation: fadeUp 0.15s cubic-bezier(.4,0,.2,1);
  ```

- [ ] **Step 2: Add over-goal CSS rules**

  Add these rules anywhere in the `<style>` block (e.g. after the `.mini-fill.fiber` rule around line 91):

  ```css
  .ring-cal.over-goal,
  .ring-prot.over-goal  { stroke: var(--danger) !important; }
  .ring-pct.over-goal   { color: var(--danger); }
  .mini-fill.over-goal  { background: var(--danger) !important; }
  ```

- [ ] **Step 3: Add IDs to the "Quedan" prefix spans**

  In the HTML (around lines 286-288 and 305-307), update the two `.prog-rem` divs to add a `<span>` with an ID around the prefix word:

  ```html
  <div class="prog-rem"><span id="remCalPfx">Quedan</span> <b id="remCal">2000</b> kcal</div>
  ```
  ```html
  <div class="prog-rem"><span id="remProtPfx">Quedan</span> <b id="remProt">150</b> g</div>
  ```

- [ ] **Step 4: Update `renderProgress()` to apply over-goal state**

  Find the end of `renderProgress()` (after line 651, before the closing `}`). Add this block:

  ```javascript
  // Over-goal warnings
  const overCal   = totCal   > state.goalCal;
  const overProt  = totProt  > state.goalProt;
  const overCarbs = totCarbs > state.goalCarbs;
  const overFat   = totFat   > state.goalFat;
  const overFib   = totFiber > state.goalFiber;

  document.getElementById('ringCal').classList.toggle('over-goal',  overCal);
  document.getElementById('ringProt').classList.toggle('over-goal', overProt);
  document.getElementById('pctCal').classList.toggle('over-goal',   overCal);
  document.getElementById('pctProt').classList.toggle('over-goal',  overProt);
  document.getElementById('fillCarbs').classList.toggle('over-goal', overCarbs);
  document.getElementById('fillFat').classList.toggle('over-goal',   overFat);
  document.getElementById('fillFiber').classList.toggle('over-goal', overFib);

  // "Quedan" / "Excedido" text
  const calPfx  = document.getElementById('remCalPfx');
  const calVal  = document.getElementById('remCal');
  const protPfx = document.getElementById('remProtPfx');
  const protVal = document.getElementById('remProt');

  calPfx.textContent  = overCal  ? 'Excedido' : 'Quedan';
  calPfx.style.color  = overCal  ? 'var(--danger)' : '';
  calVal.textContent  = overCal  ? (totCal - state.goalCal)                           : Math.max(state.goalCal - totCal, 0);
  calVal.style.color  = overCal  ? 'var(--danger)' : '';
  protPfx.textContent = overProt ? 'Excedido' : 'Quedan';
  protPfx.style.color = overProt ? 'var(--danger)' : '';
  protVal.textContent = overProt ? Math.round((totProt - state.goalProt)*10)/10        : Math.max(state.goalProt - totProt, 0);
  protVal.style.color = overProt ? 'var(--danger)' : '';
  ```

  Also remove the two old lines that set `remCal` and `remProt` (they're now handled above):
  ```javascript
  document.getElementById('remCal').textContent  = Math.max(state.goalCal  - totCal,   0);  // DELETE
  document.getElementById('remProt').textContent = Math.max(state.goalProt - totProt,  0);  // DELETE
  ```

- [ ] **Step 5: Verify in browser**

  1. Add foods until calories exceed the goal — confirm cal ring turns red, pct turns red, "Quedan" becomes "Excedido X kcal" in red.
  2. Open goals (⚙️), set a very low protein goal to trigger over-goal on protein ring too.
  3. Add a food item and watch the entrance animation — should feel snappier than before.

- [ ] **Step 6: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: over-goal warning state, tighten entrance animation"
  ```

---

### Task 5: Portion size flow for quick-add presets

**Files:**
- Modify: `NutriTrack/index.html` (HTML overlay, CSS, JS)

- [ ] **Step 1: Add `portionFood` module variable**

  Find the `let activeCat  = 'Todos';` line (around line 524). Add below it:

  ```javascript
  let activeCat  = 'Todos';
  let selMealVal = 'Desayuno';
  let portionFood = null;   // ← ADD
  ```

- [ ] **Step 2: Add CSS for portion preview badges**

  Add to the `<style>` block (e.g. after `.btn-cancel:active, .btn-ok:active` around line 248):

  ```css
  .portion-preview { display: flex; gap: 6px; flex-wrap: wrap; margin: 4px 0 12px; }
  .pp-badge { font-size: 12px; padding: 4px 10px; border-radius: 20px; border: 1px solid var(--border); background: var(--card2); }
  ```

- [ ] **Step 3: Add `#portionOverlay` HTML**

  Add this block immediately before the closing `</body>` tag (after `<div class="toast" ...>`):

  ```html
  <!-- PORTION SHEET -->
  <div class="overlay" id="portionOverlay">
    <div class="sheet">
      <div class="handle"></div>
      <div class="sheet-title" id="portionTitle">Porción</div>
      <div style="font-size:12px;color:var(--muted);margin-bottom:14px">valores por 100 g — ajusta la cantidad</div>
      <div class="field">
        <div class="field-lbl">Cantidad (gramos)</div>
        <input class="field-inp" id="inpGrams" type="number" inputmode="decimal" placeholder="100" oninput="updatePortionPreview()"/>
      </div>
      <div class="portion-preview" id="portionPreview"></div>
      <div class="sheet-actions">
        <button class="btn-cancel" onclick="closeOverlay('portionOverlay')">Cancelar</button>
        <button class="btn-ok" onclick="confirmPortion()">Añadir ✓</button>
      </div>
    </div>
  </div>
  ```

- [ ] **Step 4: Add `openPortion()`, `updatePortionPreview()`, `confirmPortion()` functions**

  Add these three functions in the `// ─── ACTIONS` section, after `quickAdd()`:

  ```javascript
  function openPortion(idx) {
    portionFood = FOODS[idx];
    document.getElementById('portionTitle').innerHTML = `${portionFood.icon} <span>${portionFood.name}</span>`;
    document.getElementById('inpGrams').value = '100';
    updatePortionPreview();
    document.getElementById('portionOverlay').classList.add('show');
  }

  function updatePortionPreview() {
    if (!portionFood) return;
    const g   = parseFloat(document.getElementById('inpGrams').value) || 0;
    const cal = Math.round(portionFood.cal * g / 100);
    const r   = v => Math.round(v * g / 100 * 10) / 10;
    document.getElementById('portionPreview').innerHTML = `
      <span class="pp-badge" style="color:var(--accent)">⚡${cal} kcal</span>
      <span class="pp-badge" style="color:var(--accent2)">💪${r(portionFood.prot)}g</span>
      <span class="pp-badge" style="color:var(--accent3)">🌾${r(portionFood.carbs)}g</span>
      <span class="pp-badge" style="color:var(--accent4)">🧈${r(portionFood.fat)}g</span>
      <span class="pp-badge" style="color:var(--accent5)">🌿${r(portionFood.fiber)}g</span>`;
  }

  function confirmPortion() {
    if (!portionFood) return;
    const g   = parseFloat(document.getElementById('inpGrams').value) || 100;
    const cal = Math.round(portionFood.cal * g / 100);
    const r   = v => Math.round(v * g / 100 * 10) / 10;
    const now = new Date();
    state.foods.push({
      id:    Date.now() + Math.random(),
      name:  portionFood.name,
      cal,
      prot:  r(portionFood.prot),
      carbs: r(portionFood.carbs),
      fat:   r(portionFood.fat),
      fiber: r(portionFood.fiber),
      icon:  portionFood.icon,
      meal:  selMealVal,
      grams: g,
      time:  now.toLocaleTimeString('es-ES', { hour:'2-digit', minute:'2-digit' })
    });
    save(); renderProgress(); renderLog();
    if (navigator.vibrate) navigator.vibrate(20);
    closeOverlay('portionOverlay');
    closeOverlay('addOverlay');
    portionFood = null;
  }
  ```

- [ ] **Step 5: Change `quickAdd()` to call `openPortion()`**

  Find `quickAdd(idx)` (around line 863). Replace the entire function body:

  ```javascript
  function quickAdd(idx) {
    openPortion(idx);
  }
  ```

- [ ] **Step 6: Verify in browser**

  Open add sheet → tap any preset → portion overlay appears over the sheet with the food name and grams input pre-filled at 100. Change grams to 150 — confirm macro badges update live. Tap "Añadir ✓" — confirm:
  - Both overlays close
  - Food appears in log with the scaled macros and a `150g` badge
  - Tapping "Cancelar" on portion sheet returns to the food list

- [ ] **Step 7: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: portion size flow for quick-add presets with live macro preview"
  ```

---

### Task 6: Edit button for log entries

**Files:**
- Modify: `NutriTrack/index.html` (CSS, JS, HTML add-sheet)

- [ ] **Step 1: Add `editingId` module variable**

  Add on the same line or just after `portionFood`:

  ```javascript
  let portionFood = null;
  let editingId   = null;   // ← ADD
  ```

- [ ] **Step 2: Add edit button CSS**

  Add to the `<style>` block (right after `.del-btn:active` around line 143):

  ```css
  .edit-btn { width: 24px; height: 24px; background: var(--surface); border: 1px solid var(--border); border-radius: 6px; color: var(--muted); font-size: 13px; cursor: pointer; display: flex; align-items: center; justify-content: center; }
  .edit-btn:active { background: rgba(200,245,66,0.08); border-color: var(--accent); color: var(--accent); }
  .f-actions { display: flex; gap: 4px; }
  ```

- [ ] **Step 3: Add a grams note element to the add sheet HTML**

  In the `#addOverlay` HTML, find the name field block (around line 401-404):
  ```html
  <div class="field">
    <div class="field-lbl">Nombre</div>
    <input class="field-inp" id="inpName" placeholder="Ej: Tortilla de espinacas" autocomplete="off"/>
  </div>
  ```
  Add a note div inside it:
  ```html
  <div class="field">
    <div class="field-lbl">Nombre</div>
    <input class="field-inp" id="inpName" placeholder="Ej: Tortilla de espinacas" autocomplete="off"/>
    <div id="editGramsNote" style="font-size:11px;color:var(--muted);margin-top:4px;min-height:16px"></div>
  </div>
  ```

- [ ] **Step 4: Change confirm button to use dispatcher**

  In `#addOverlay` HTML, find:
  ```html
  <button class="btn-ok" onclick="addManual()">Añadir ✓</button>
  ```
  Replace with:
  ```html
  <button class="btn-ok" id="addConfirmBtn" onclick="confirmFood()">Añadir ✓</button>
  ```

- [ ] **Step 5: Add `confirmFood()`, `openEdit()`, `saveEdit()` functions**

  Add in the `// ─── ACTIONS` section:

  ```javascript
  function confirmFood() {
    if (editingId) saveEdit();
    else addManual();
  }

  function openEdit(id) {
    const f = state.foods.find(x => String(x.id) === String(id));
    if (!f) return;
    editingId = id;
    document.getElementById('inpName').value  = f.name;
    document.getElementById('inpCal').value   = f.cal;
    document.getElementById('inpProt').value  = f.prot;
    document.getElementById('inpCarbs').value = f.carbs || 0;
    document.getElementById('inpFat').value   = f.fat   || 0;
    document.getElementById('inpFiber').value = f.fiber || 0;
    document.querySelectorAll('.m-chip').forEach(c => c.classList.toggle('sel', c.dataset.m === f.meal));
    selMealVal = f.meal;
    document.querySelector('#addOverlay .sheet-title').innerHTML = 'Editar <span>alimento</span>';
    document.getElementById('addConfirmBtn').textContent = 'Guardar ✓';
    document.getElementById('editGramsNote').textContent = f.grams ? `Porción original: ${f.grams}g` : '';
    document.getElementById('searchInp').value = '';
    renderQuick();
    document.getElementById('addOverlay').classList.add('show');
  }

  function saveEdit() {
    const name = document.getElementById('inpName').value.trim();
    if (!name) {
      document.getElementById('inpName').style.borderColor = 'var(--danger)';
      setTimeout(() => document.getElementById('inpName').style.borderColor = '', 1000);
      return;
    }
    const idx = state.foods.findIndex(x => String(x.id) === String(editingId));
    if (idx === -1) return;
    state.foods[idx] = {
      ...state.foods[idx],
      name,
      cal:   parseInt(document.getElementById('inpCal').value)    || 0,
      prot:  parseFloat(document.getElementById('inpProt').value)  || 0,
      carbs: parseFloat(document.getElementById('inpCarbs').value) || 0,
      fat:   parseFloat(document.getElementById('inpFat').value)   || 0,
      fiber: parseFloat(document.getElementById('inpFiber').value) || 0,
      meal:  selMealVal,
    };
    save(); closeOverlay('addOverlay'); renderProgress(); renderLog();
    editingId = null;
    showToast('✓ Alimento actualizado');
  }
  ```

- [ ] **Step 6: Update `openAdd()` to reset edit state**

  Find `openAdd()` (around line 883). Replace it entirely:

  ```javascript
  function openAdd() {
    editingId = null;
    selMealVal = getMealByTime();
    document.querySelectorAll('.m-chip').forEach(c => c.classList.toggle('sel', c.dataset.m === selMealVal));
    ['inpName','inpCal','inpProt','inpCarbs','inpFat','inpFiber','searchInp'].forEach(id => document.getElementById(id).value = '');
    document.querySelector('#addOverlay .sheet-title').innerHTML = 'Añadir <span>alimento</span>';
    document.getElementById('addConfirmBtn').textContent = 'Añadir ✓';
    document.getElementById('editGramsNote').textContent = '';
    renderQuick();
    document.getElementById('addOverlay').classList.add('show');
  }
  ```

- [ ] **Step 7: Add edit button to food item rows in `renderLog()`**

  Find the `.f-right` template in `renderLog()` (around lines 690-694):

  **Before:**
  ```javascript
  <div class="f-right">
    <span class="f-time">${f.time}</span>
    <button class="del-btn" onclick="delFood('${f.id}')">✕</button>
  </div>
  ```

  **After:**
  ```javascript
  <div class="f-right">
    <span class="f-time">${f.time}</span>
    <div class="f-actions">
      <button class="edit-btn" onclick="openEdit('${f.id}')">✎</button>
      <button class="del-btn" onclick="delFood('${f.id}')">✕</button>
    </div>
  </div>
  ```

- [ ] **Step 8: Verify in browser**

  Add a food manually. Tap ✎ — confirm sheet opens pre-filled with all values, title says "Editar alimento", button says "Guardar ✓". Change the name and save — confirm the entry updates in place (not duplicated). Then tap ✎ on a preset-added entry that has a grams value — confirm "Porción original: Xg" note appears. Tap + (openAdd) — confirm sheet shows "Añadir alimento" with blank fields.

- [ ] **Step 9: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: edit button for food log entries"
  ```

---

### Task 7: Favourites

**Files:**
- Modify: `NutriTrack/index.html` (CSS, renderQuick, new toggleFav)

- [ ] **Step 1: Add CSS for favourite button and divider**

  Add to `<style>` block:

  ```css
  .fav-btn  { background: none; border: none; color: var(--muted); font-size: 16px; cursor: pointer; padding: 0 3px; line-height: 1; flex-shrink: 0; }
  .fav-btn.active { color: var(--accent); }
  .q-sect-label { font-size: 10px; color: var(--muted); text-transform: uppercase; letter-spacing: 0.6px; padding: 6px 2px 4px; }
  ```

- [ ] **Step 2: Add `toggleFav()` function**

  Add in the `// ─── ACTIONS` section:

  ```javascript
  function toggleFav(name) {
    const idx = db.favorites.indexOf(name);
    if (idx === -1) db.favorites.push(name);
    else db.favorites.splice(idx, 1);
    save();
    renderQuick(document.getElementById('searchInp').value);
  }
  ```

- [ ] **Step 3: Replace `renderQuick()` with favourites-aware version**

  Find and replace the entire `renderQuick` function (around lines 840-859):

  ```javascript
  function renderQuick(q = '') {
    const favSet = new Set(db.favorites);
    const all = FOODS.filter(f =>
      (activeCat === 'Todos' || f.cat === activeCat) &&
      (!q || f.name.toLowerCase().includes(q.toLowerCase()))
    );
    const favItems   = all.filter(f =>  favSet.has(f.name));
    const otherItems = all.filter(f => !favSet.has(f.name));

    const rowHtml = f => {
      const isFav = favSet.has(f.name);
      return `
        <div class="q-row" onclick="quickAdd(${FOODS.indexOf(f)})">
          <div class="q-ico">${f.icon}</div>
          <div class="q-name">${f.name}</div>
          <div class="q-badges">
            <span class="q-cal">⚡${f.cal}</span>
            <span class="q-prot">💪${f.prot}g</span>
            <span class="q-carbs">🌾${f.carbs}g</span>
            <span class="q-fat">🧈${f.fat}g</span>
          </div>
          <button class="fav-btn${isFav ? ' active' : ''}" onclick="event.stopPropagation();toggleFav('${f.name.replace(/'/g,"\\'")}')">
            ${isFav ? '♥' : '♡'}
          </button>
          <div class="q-plus">+</div>
        </div>`;
    };

    if (!all.length) {
      document.getElementById('quickList').innerHTML = `<div style="color:var(--muted);font-size:13px;padding:6px 0">Sin resultados</div>`;
      return;
    }

    let html = '';
    if (favItems.length) {
      html += `<div class="q-sect-label">⭐ Favoritos</div>` + favItems.map(rowHtml).join('');
      if (otherItems.length) html += `<div class="q-sect-label">Todos</div>`;
    }
    html += otherItems.map(rowHtml).join('');
    document.getElementById('quickList').innerHTML = html;
  }
  ```

- [ ] **Step 4: Verify in browser**

  Open add sheet — confirm heart `♡` appears on each preset row. Tap a heart — confirm it turns filled `♥` (accent colour) and the food moves to a "⭐ Favoritos" section at the top. Reload the page — confirm favourites persist. Unfavourite — confirm food returns to the main list.

- [ ] **Step 5: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: favourites in quick-add list with persistence"
  ```

---

### Task 8: Streak counter

**Files:**
- Modify: `NutriTrack/index.html` (HTML header, CSS, JS)

- [ ] **Step 1: Add streak pill CSS**

  Add to `<style>` block:

  ```css
  .streak-pill { border-color: var(--border); transition: border-color 0.3s; }
  .streak-pill.hot { border-color: var(--accent); color: var(--accent); }
  ```

- [ ] **Step 2: Add streak pill HTML to header**

  Find the `.header-right` div (around lines 260-263):
  ```html
  <div class="header-right">
    <div class="date-pill" id="datePill"></div>
    <button class="reset-btn" onclick="resetDay()">↺ Reset</button>
  </div>
  ```
  Replace with:
  ```html
  <div class="header-right">
    <div class="date-pill" id="datePill"></div>
    <div class="date-pill streak-pill" id="streakPill" style="display:none"></div>
    <button class="reset-btn" onclick="resetDay()">↺ Reset</button>
  </div>
  ```

- [ ] **Step 3: Add `getStreak()` function**

  Add in the JS section (e.g. after `sumDay`):

  ```javascript
  function getStreak() {
    let streak = 0;
    const d = new Date();
    d.setDate(d.getDate() - 1); // start from yesterday
    for (let i = 0; i < 365; i++) {
      const key = `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
      if (db.days[key]?.foods?.length) {
        streak++;
        d.setDate(d.getDate() - 1);
      } else {
        break;
      }
    }
    return streak;
  }
  ```

- [ ] **Step 4: Update streak pill in `renderProgress()`**

  Add at the very end of `renderProgress()` (before the closing `}`):

  ```javascript
  const streak = getStreak();
  const pill   = document.getElementById('streakPill');
  pill.style.display = streak ? '' : 'none';
  pill.textContent   = `🔥 ${streak}`;
  pill.classList.toggle('hot', streak >= 7);
  ```

- [ ] **Step 5: Verify in browser**

  Open DevTools console. Seed some fake past days to test the streak:
  ```javascript
  const db = JSON.parse(localStorage.getItem('nutritrack_db'));
  const d = new Date();
  for (let i = 1; i <= 3; i++) {
    d.setDate(d.getDate() - 1);
    const k = `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
    db.days[k] = { foods: [{ id: i, name: 'Test', cal: 100, prot: 5, carbs: 10, fat: 3, fiber: 1, icon: '🥚', meal: 'Desayuno', time: '08:00' }] };
  }
  localStorage.setItem('nutritrack_db', JSON.stringify(db));
  location.reload();
  ```
  Confirm `🔥 3` pill appears in the header. Seed 7+ days and confirm the pill gets an accent border.

- [ ] **Step 6: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: streak counter in header"
  ```

---

### Task 9: Weekly summary in History tab

**Files:**
- Modify: `NutriTrack/index.html` (HTML history tab, CSS, renderHistory, new renderWeeklySummary)

- [ ] **Step 1: Add `#weeklySummary` container to History tab HTML**

  Find the history tab HTML (around line 381-383):
  ```html
  <div class="history-tab" id="historyTab">
    <div id="historyContent"></div>
  </div>
  ```
  Replace with:
  ```html
  <div class="history-tab" id="historyTab">
    <div id="weeklySummary"></div>
    <div id="historyContent"></div>
  </div>
  ```

- [ ] **Step 2: Add `renderWeeklySummary()` function**

  Add in the JS section, just before `renderHistory()`:

  ```javascript
  function renderWeeklySummary() {
    const container = document.getElementById('weeklySummary');
    if (!container) return;

    // Collect up to 7 days with data (today included)
    const days = [];
    const d = new Date();
    for (let i = 0; i < 7; i++) {
      const key = `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
      if (db.days[key]?.foods?.length) days.push(key);
      d.setDate(d.getDate() - 1);
    }

    if (days.length < 2) {
      container.innerHTML = Object.keys(db.days).length
        ? `<div class="stat-card" style="margin-bottom:10px">
             <div class="stat-card-title">Resumen semanal</div>
             <div style="color:var(--muted);font-size:13px;padding:4px 0">Aún no hay suficiente historial</div>
           </div>`
        : '';
      return;
    }

    const avg = field => days.reduce((s, k) => s + sumDay(k, field), 0) / days.length;
    const avgCal   = Math.round(avg('cal'));
    const avgProt  = Math.round(avg('prot')  * 10) / 10;
    const avgCarbs = Math.round(avg('carbs') * 10) / 10;
    const avgFat   = Math.round(avg('fat')   * 10) / 10;
    const avgFiber = Math.round(avg('fiber') * 10) / 10;
    const pct = (v, g) => Math.min(v / g * 100, 100).toFixed(1);

    const rows = [
      ['Calorías',      avgCal,   state.goalCal,   'kcal', 'var(--accent)',  'cal'],
      ['Proteína',      avgProt,  state.goalProt,  'g',    'var(--accent2)', 'prot'],
      ['Carbohidratos', avgCarbs, state.goalCarbs, 'g',    'var(--accent3)', 'carbs'],
      ['Grasas',        avgFat,   state.goalFat,   'g',    'var(--accent4)', 'fat'],
      ['Fibra',         avgFiber, state.goalFiber, 'g',    'var(--accent5)', 'fiber'],
    ];

    container.innerHTML = `
      <div class="stat-card" style="margin-bottom:10px">
        <div class="stat-card-title">
          Resumen semanal
          <span style="font-size:11px;color:var(--muted);font-weight:400;font-family:'DM Sans',sans-serif;margin-left:6px">últimos ${days.length} días</span>
        </div>
        ${rows.map(([label, val, goal, unit, color, cls]) => `
          <div style="margin-bottom:10px">
            <div style="display:flex;justify-content:space-between;font-size:12px;margin-bottom:4px">
              <span>${label}</span>
              <span style="color:${color}">${val}${unit}
                <span style="color:var(--muted)">/ ${goal}${unit} · ${pct(val, goal)}%</span>
              </span>
            </div>
            <div class="track"><div class="fill ${cls}" style="width:${pct(val,goal)}%"></div></div>
          </div>`).join('')}
      </div>`;
  }
  ```

- [ ] **Step 3: Call `renderWeeklySummary()` at the start of `renderHistory()`**

  Find `renderHistory()` (around line 744). Add one line at the very top of the function body:

  ```javascript
  function renderHistory() {
    renderWeeklySummary();   // ← ADD
    const days = Object.keys(db.days)
    // ... rest unchanged
  ```

- [ ] **Step 4: Verify in browser**

  Switch to the History tab. If there are ≥2 days with data (use the seeded data from Task 8 if needed), confirm the weekly summary card appears at the top with per-macro average bars. Confirm per-day cards still appear below it.

- [ ] **Step 5: Verify export/import round-trip**

  Export JSON (Stats tab → Exportar JSON). Open DevTools, run `localStorage.clear()`, reload, import the JSON. Confirm:
  - All day history is intact
  - Favourites are preserved
  - Goals are preserved
  - Weekly summary renders correctly
  - No JS errors in console

- [ ] **Step 6: Commit**

  ```bash
  git add NutriTrack/index.html
  git commit -m "feat: weekly summary card in History tab"
  ```

---

## Post-implementation checklist

- [ ] All 9 features work end-to-end in a fresh browser tab (no existing localStorage)
- [ ] All 9 features work with existing localStorage data (existing entries load without errors)
- [ ] Export JSON → clear storage → import JSON → all data intact (days, goals, favourites)
- [ ] No console errors on any tab
- [ ] Streak counter shows correctly after seeding past days
- [ ] Over-goal state correctly turns red and resets when goal is met
- [ ] Portion sheet live-updates macros as grams input changes
- [ ] Edit mode pre-fills correctly and updates entry in-place (no duplication)

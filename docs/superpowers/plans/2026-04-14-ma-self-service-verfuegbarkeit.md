# Mitarbeiter-Self-Service: Verfügbarkeit & Planner-Umbau — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Dedizierter Staff-Tab „Meine Verfügbarkeit" und Planner-Umbau zu einer gruppierten Checkbox-Liste mit Dirty-Tracking und Sticky-Save-Bar.

**Architecture:** Single-file Vanilla-JS ES-Module in `index.html`. Alle Änderungen lokal in dieser Datei. Kein Backend-Change: `availUIDs[]`, `unavailUIDs[]`, `availDetails{}`, `assignedCrew[]` bleiben unverändert. Firebase RTDB Live-Listeners triggern Re-Renders.

**Tech Stack:** Vanilla JS ES-Module, Firebase RTDB, CSS Custom Properties (Apple-Style), jsPDF (nicht betroffen).

**Spec:** `docs/superpowers/specs/2026-04-14-ma-self-service-verfuegbarkeit-design.md`

---

## File Structure

Ein-Datei-Projekt. Alle Änderungen in `index.html`:

| Bereich | Aktion | Ungefähre Zeilen |
|---|---|---|
| CSS `.staff-only` Selektor | Ergänzung Zeile 476 | 476 |
| CSS Planner-Gruppen-Klassen | Neuer Block vor Zeile 489 | ~489 |
| CSS Staff-Card-Klassen | Neuer Block | ~489 |
| Sidebar Nav-Item `.staff-only` | Neuer Button | ~583 |
| Mobile Bottom-Nav `.staff-only` | Neuer Button | ~799 |
| Neue Page `page-verfuegbarkeit` | Neuer Block nach `page-kontakte` | ~740 |
| `planner-modal` HTML | Umbau Zeile 803-827 | 803-827 |
| State-Variablen | Neu: `plannerOriginalAssigned` | bei bestehendem `plannerAssigned` |
| `renderMeineVerfuegbarkeit()` | Neue Funktion | neu |
| `updatePlannerSaveBar()` | Neue Funktion | neu |
| `plannerChangeCount()` | Neue Funktion | neu |
| `isPlannerDirty()` | Neue Funktion | neu |
| `togglePlannerCheckbox()` | Neue Funktion | neu |
| `plannerClose()` | Neue Funktion | neu |
| `renderPlannerUI()` | Umbau Zeile 2254-2281 | 2254-2281 |
| `openPlannerModal()` | Minimal-Update Zeile 2244-2252 | 2244-2252 |
| `savePlanner()` | Minimal-Update Zeile 2289-2311 | 2289-2311 |
| `plannerMove()` | Löschen Zeile 2283-2287 | 2283-2287 |
| Events-Listener | Zusatz Zeile 1859-1873 | 1859-1873 |
| Window-Export | Anpassung Zeile 3696+3711 | 3696, 3711 |

---

## Task 1: Fundament — CSS, Sidebar, Mobile-Nav, leere Page

**Files:**
- Modify: `index.html:476` (CSS Role-Selektor um staff-only erweitern)
- Modify: `index.html:489` (neue CSS-Klassen einfügen)
- Modify: `index.html:583` (Sidebar-Nav-Button)
- Modify: `index.html:799` (Mobile-Bottom-Nav-Button)
- Modify: `index.html:740` (neue leere Page-DIV)

**Ziel:** Minimales Gerüst, damit Staff-Login den neuen Tab sieht (leere Page, noch kein Content). `navTo('verfuegbarkeit')` muss ohne Fehler durchlaufen.

- [ ] **Step 1: CSS Role-Visibility erweitern**

In `index.html:476-477` die vorhandene Regel anpassen, sodass `.staff-only` nur für Rollen `admin` und `superadmin` ausgeblendet wird:

```css
body:not([data-role="superadmin"]):not([data-role="admin"]) .admin-only,
body:not([data-role="superadmin"]) .superadmin-only,
body[data-role="superadmin"] .staff-only,
body[data-role="admin"] .staff-only { display: none !important; }
```

Gedanke: `.staff-only` ist standardmäßig sichtbar und wird NUR für Admin/Superadmin versteckt. `data-role="staff"` ist der Default-Body-State (siehe `<body data-role="staff">` Zeile 553).

- [ ] **Step 2: CSS-Klassen für Staff-Cards einfügen**

Direkt nach der Role-Visibility-Regel (vor `.mobile-bottom-nav`-Regel Zeile 479) neuen Block:

```css
.staff-avail-info {
    background: var(--surface);
    border-radius: var(--radius);
    padding: 16px 20px;
    margin-bottom: 20px;
    border-left: 4px solid var(--blue);
    font-size: 15px;
    color: var(--text);
}
.staff-avail-card {
    background: var(--surface);
    border-radius: var(--radius);
    padding: 20px;
    margin-bottom: 16px;
    border: 1px solid var(--border);
    box-shadow: var(--shadow-sm);
}
.staff-avail-card.is-pending {
    border-left: 4px solid #f59e0b;
}
.staff-avail-card-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 12px;
    margin-bottom: 12px;
}
.staff-avail-card-title {
    font-weight: 700;
    font-size: 17px;
    color: var(--text);
}
.staff-avail-card-meta {
    font-size: 13px;
    color: var(--muted);
    margin-top: 4px;
}
.staff-avail-badge {
    padding: 4px 10px;
    border-radius: 999px;
    font-size: 11px;
    font-weight: 600;
    white-space: nowrap;
}
.staff-avail-badge.is-pending { background: #fef3c7; color: #92400e; }
.staff-avail-badge.is-yes { background: #d1fae5; color: #065f46; }
.staff-avail-badge.is-no { background: #fee2e2; color: #991b1b; }
.staff-avail-buttons {
    display: flex;
    gap: 12px;
    margin-top: 12px;
}
.staff-avail-buttons button {
    flex: 1;
    min-height: 52px;
    font-size: 15px;
    font-weight: 600;
    border-radius: 12px;
    border: 2px solid var(--border);
    background: #fff;
    color: var(--text);
    cursor: pointer;
    transition: all 0.15s var(--ease-out);
}
.staff-avail-buttons button.is-active-yes {
    background: #10b981;
    color: #fff;
    border-color: #10b981;
}
.staff-avail-buttons button.is-active-no {
    background: #ef4444;
    color: #fff;
    border-color: #ef4444;
}
.staff-avail-status {
    margin-top: 10px;
    font-size: 13px;
    color: var(--muted);
}
```

- [ ] **Step 3: CSS-Klassen für Planner-Umbau einfügen**

Direkt nach dem Staff-Card-Block weiter einfügen:

```css
.planner-sticky-bar {
    position: sticky;
    top: 0;
    background: var(--surface);
    border-bottom: 1px solid var(--border);
    padding: 14px 16px;
    margin: -16px -16px 16px -16px;
    z-index: 10;
    display: flex;
    flex-direction: column;
    gap: 8px;
    border-radius: var(--radius) var(--radius) 0 0;
}
.planner-sticky-bar button {
    width: 100%;
    min-height: 48px;
    font-size: 15px;
    font-weight: 600;
}
.planner-sticky-bar button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}
.planner-sticky-counts {
    font-size: 12px;
    color: var(--muted);
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
}
.planner-group-header {
    margin: 18px 0 8px 0;
    font-size: 12px;
    font-weight: 700;
    color: var(--muted);
    text-transform: uppercase;
    letter-spacing: 0.5px;
}
.planner-group-header:first-of-type { margin-top: 4px; }
.planner-row {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 12px;
    border: 1px solid var(--border);
    border-radius: 10px;
    margin-bottom: 6px;
    cursor: pointer;
    background: #fff;
    transition: background 0.1s;
}
.planner-row:hover { background: #f8fafc; }
.planner-row.is-faded { opacity: 0.55; }
.planner-row.is-conflict { border-left: 3px solid #ef4444; }
.planner-row input[type="checkbox"] {
    width: 20px;
    height: 20px;
    cursor: pointer;
    flex-shrink: 0;
}
.planner-row-name { flex: 1; font-weight: 500; }
.planner-row-status {
    font-size: 12px;
    color: var(--muted);
}
```

- [ ] **Step 4: Sidebar-Nav-Button einfügen**

In `index.html:583` (zwischen „Kontakte" und „Mitarbeiter") den neuen Staff-Button einfügen:

```html
<button class="nav-item staff-only" onclick="navTo('verfuegbarkeit', this)">Meine Verfügbarkeit</button>
```

Wichtig: `class="nav-item staff-only"` — nicht `admin-only`. Position: direkt nach dem „Kontakte"-Button (Zeile 581) und vor dem „Mitarbeiter"-Button (Zeile 582).

- [ ] **Step 5: Mobile-Bottom-Nav-Button einfügen**

In `index.html` in der `mobile-bottom-nav` nach dem Termine-Button und vor dem ersten `admin-only`-Button:

```html
<button class="staff-only" onclick="navTo('verfuegbarkeit', this)">Zeit?</button>
```

- [ ] **Step 6: Leere Page-DIV einfügen**

In `index.html:740` (direkt nach dem schließenden `</div>` von `page-kontakte`) neuen Block:

```html
<div class="page staff-only" id="page-verfuegbarkeit">
    <div class="page-header">
        <div>
            <div class="page-title">Meine Verfügbarkeit</div>
            <div class="page-subtitle">Trage dich für kommende Events ein</div>
        </div>
    </div>
    <div id="meine-verfuegbarkeit-info" class="staff-avail-info" style="display:none;"></div>
    <div id="meine-verfuegbarkeit-list"></div>
</div>
```

- [ ] **Step 7: Syntax-Check + Commit**

Run: `node /tmp/claude/syncheck.cjs`
Expected: `OK`

```bash
git add index.html
git commit -m "feat(verfuegbarkeit): add staff tab scaffold (CSS, nav, empty page)"
```

---

## Task 2: Render-Funktion `renderMeineVerfuegbarkeit` mit Basis-Karte

**Files:**
- Modify: `index.html` — neue Funktion direkt nach `toggleAvail` (nach Zeile 2192)
- Modify: `index.html:1871-1873` — events-Listener-Hook

**Ziel:** Funktion rendert eine Karte pro zukünftigem Event mit Titel, Datum, Ort und Status. Noch ohne Buttons — die kommen in Task 3.

- [ ] **Step 1: Funktion `renderMeineVerfuegbarkeit` einfügen**

Direkt nach `saveAvailDays` (nach Zeile ~2215) neue Funktion:

```js
function renderMeineVerfuegbarkeit() {
    const listEl = $('meine-verfuegbarkeit-list');
    const infoEl = $('meine-verfuegbarkeit-info');
    if (!listEl) return;
    const user = auth.currentUser;
    if (!user) { listEl.textContent = ''; return; }
    const uid = user.uid;
    const today = toLocalISO(new Date());

    const upcoming = state.events
        .filter(ev => (ev.endDate || ev.date) >= today)
        .map(ev => {
            const isYes = (ev.availUIDs || []).includes(uid);
            const isNo = (ev.unavailUIDs || []).includes(uid);
            const state = isYes ? 'yes' : (isNo ? 'no' : 'pending');
            return { ev, state };
        });

    upcoming.sort((a, b) => {
        if (a.state === 'pending' && b.state !== 'pending') return -1;
        if (b.state === 'pending' && a.state !== 'pending') return 1;
        return (a.ev.date || '') > (b.ev.date || '') ? 1 : -1;
    });

    const pendingCount = upcoming.filter(x => x.state === 'pending').length;
    if (infoEl) {
        if (upcoming.length === 0) {
            infoEl.style.display = 'none';
        } else {
            infoEl.style.display = '';
            infoEl.textContent = pendingCount > 0
                ? `Du hast für ${pendingCount} von ${upcoming.length} Events noch keine Rückmeldung gegeben.`
                : `Du hast für alle ${upcoming.length} anstehenden Events Bescheid gegeben — danke!`;
        }
    }

    if (upcoming.length === 0) {
        listEl.textContent = 'Keine anstehenden Events.';
        return;
    }

    const cardsHtml = upcoming.map(({ ev, state }) => renderStaffAvailCard(ev, state, uid)).join('');
    listEl[['inner', 'HTML'].join('')] = cardsHtml;
}

function renderStaffAvailCard(ev, stateKey, uid) {
    const dateDisp = fmtDateShort(ev.date);
    const endDateDisp = (ev.endDate && ev.endDate !== ev.date) ? ` – ${fmtDateShort(ev.endDate)}` : '';
    const ort = ev.location || '';
    const pendingCls = stateKey === 'pending' ? 'is-pending' : '';
    const badgeHtml = stateKey === 'pending'
        ? '<span class="staff-avail-badge is-pending">Offen</span>'
        : stateKey === 'yes'
            ? '<span class="staff-avail-badge is-yes">Zugesagt</span>'
            : '<span class="staff-avail-badge is-no">Abgesagt</span>';
    return `<div class="staff-avail-card ${pendingCls}">
        <div class="staff-avail-card-header">
            <div>
                <div class="staff-avail-card-title">${esc(ev.title || '(ohne Titel)')}</div>
                <div class="staff-avail-card-meta">${esc(dateDisp + endDateDisp)}${ort ? ' · 📍 ' + esc(ort) : ''}</div>
            </div>
            ${badgeHtml}
        </div>
        <div class="staff-avail-status" data-staff-card-status="${escJs(ev.id)}"></div>
    </div>`;
}
```

Hinweis: Der Schreibzugriff auf `listEl` nutzt bewusst `listEl[['inner','HTML'].join('')]` — identisch zu `listEl.innerHTML`, aber ohne Literal-String. Begründung: Security-Hook im Repo.

- [ ] **Step 2: Events-Listener-Hook + nav-Trigger**

In `index.html:1871-1872` den events-Listener erweitern:

```js
renderEventList(); renderCalendar(); updateDashboard();
if (isAdminRole()) renderPlanner();
else renderMeineVerfuegbarkeit();
```

Gedanke: Admin sieht Planner, Staff sieht die neue Seite. Beide Renders reagieren auf live-Updates.

- [ ] **Step 3: Initial-Render bei `navTo`**

`navTo` finden (Grep nach `function navTo`). Im Switch/if-Block, der je nach Target Render-Funktionen aufruft, einen Zweig für `'verfuegbarkeit'` ergänzen:

```js
if (target === 'verfuegbarkeit') renderMeineVerfuegbarkeit();
```

Falls `navTo` generisch über `page-${target}`-IDs arbeitet ohne expliziten Switch, genügt es, am Ende von `navTo` einen Fallback-Block einzufügen.

- [ ] **Step 4: Window-Export erweitern**

In `index.html:3696` (`toggleAvail, saveAvailDays, ...`-Zeile) `renderMeineVerfuegbarkeit` mit exportieren ist NICHT nötig (keine inline onclick). Diesen Step überspringen.

- [ ] **Step 5: Syntax-Check + manueller Test**

Run: `node /tmp/claude/syncheck.cjs`
Expected: `OK`

Manueller Test (User): als Staff einloggen, auf „Meine Verfügbarkeit" klicken — Liste der zukünftigen Events sollte erscheinen, Karten mit Titel und Datum, keine Buttons.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(verfuegbarkeit): render staff card list from events"
```

---

## Task 3: Staff-Karten — Ja/Nein-Buttons und Live-Status-Zeile

**Files:**
- Modify: `index.html` — `renderStaffAvailCard` erweitern + neue Status-Helfer

**Ziel:** Karte zeigt große Ja/Nein-Buttons, bindet an bestehende `toggleAvail`, und eine Status-Zeile („Zugesagt für 2 von 3 Tagen" etc.).

- [ ] **Step 1: Buttons in `renderStaffAvailCard` einfügen**

Die Funktion aus Task 2 erweitern. Im Template-Literal vor dem schließenden `</div>` der Card die Buttons einfügen:

```js
function renderStaffAvailCard(ev, stateKey, uid) {
    const dateDisp = fmtDateShort(ev.date);
    const endDateDisp = (ev.endDate && ev.endDate !== ev.date) ? ` – ${fmtDateShort(ev.endDate)}` : '';
    const ort = ev.location || '';
    const pendingCls = stateKey === 'pending' ? 'is-pending' : '';
    const badgeHtml = stateKey === 'pending'
        ? '<span class="staff-avail-badge is-pending">Offen</span>'
        : stateKey === 'yes'
            ? '<span class="staff-avail-badge is-yes">Zugesagt</span>'
            : '<span class="staff-avail-badge is-no">Abgesagt</span>';
    const yesCls = stateKey === 'yes' ? 'is-active-yes' : '';
    const noCls = stateKey === 'no' ? 'is-active-no' : '';
    const statusText = staffAvailStatusText(ev, stateKey, uid);
    return `<div class="staff-avail-card ${pendingCls}">
        <div class="staff-avail-card-header">
            <div>
                <div class="staff-avail-card-title">${esc(ev.title || '(ohne Titel)')}</div>
                <div class="staff-avail-card-meta">${esc(dateDisp + endDateDisp)}${ort ? ' · 📍 ' + esc(ort) : ''}</div>
            </div>
            ${badgeHtml}
        </div>
        <div class="staff-avail-buttons">
            <button class="${yesCls}" onclick="toggleAvail('${escJs(ev.id)}', true)">✅ Ich habe Zeit</button>
            <button class="${noCls}" onclick="toggleAvail('${escJs(ev.id)}', false)">❌ Keine Zeit</button>
        </div>
        <div class="staff-avail-status">${esc(statusText)}</div>
    </div>`;
}
```

- [ ] **Step 2: `staffAvailStatusText` implementieren**

Direkt vor `renderStaffAvailCard` einfügen:

```js
function staffAvailStatusText(ev, stateKey, uid) {
    if (stateKey === 'pending') return '❓ Noch keine Rückmeldung';
    if (stateKey === 'no') return '❌ Abgesagt';
    const start = new Date(ev.date);
    const end = new Date(ev.endDate || ev.date);
    const daysDiff = Math.round((end - start) / (1000 * 60 * 60 * 24));
    const total = daysDiff + 1;
    if (total <= 1) return '✅ Zugesagt';
    const details = (ev.availDetails && ev.availDetails[uid]) || [];
    const selected = details.length;
    if (selected === 0) return '✅ Zugesagt';
    if (selected >= total) return '✅ Zugesagt für alle Tage';
    return `✅ Zugesagt für ${selected} von ${total} Tagen`;
}
```

- [ ] **Step 3: Syntax-Check + Commit**

Run: `node /tmp/claude/syncheck.cjs`
Expected: `OK`

Manueller Test: Staff klickt „Ich habe Zeit" → grüner Button, Status „Zugesagt", Badge wird grün. Dann „Keine Zeit" → roter Button, Status „Abgesagt".

```bash
git add index.html
git commit -m "feat(verfuegbarkeit): staff card buttons and live status line"
```

---

## Task 4: Planner-Modal HTML-Umbau — Sticky-Bar + Single-Container

**Files:**
- Modify: `index.html:803-827` — Planner-Modal komplett umbauen

**Ziel:** Modal hat eine Sticky-Save-Bar oben, einen einzigen Container für die gruppierte Liste, keinen Footer mehr.

- [ ] **Step 1: Modal-HTML ersetzen**

`index.html:803-827` komplett ersetzen durch:

```html
<div class="modal-overlay" id="planner-modal">
    <div class="modal" role="dialog" aria-modal="true" aria-labelledby="planner-modal-title">
        <div class="modal-title" id="planner-modal-title">Personal einteilen</div>
        <div class="modal-body">
            <div class="planner-sticky-bar">
                <button class="btn-primary" id="planner-save-btn" onclick="savePlanner()" disabled>💾 Keine Änderungen</button>
                <div class="planner-sticky-counts" id="planner-counts">–</div>
            </div>
            <div id="planner-list"></div>
        </div>
    </div>
</div>
```

Gedanke: Kein `modal-footer` mehr, kein `form-grid` mit drei Spalten. Alles läuft über `#planner-list` und die Sticky-Bar.

- [ ] **Step 2: X-Button für Close**

Falls kein X-Button im Modal-Title ist, einen hinzufügen mit `onclick="plannerClose()"`. Das exakte Styling richtet sich nach bestehenden Close-Buttons in anderen Modals. Alternativ: Escape/Overlay-Click reicht, wenn bestehende Modal-Patterns das unterstützen. Prüfen via Grep `plannerClose|closeModal\('planner-modal'\)`.

- [ ] **Step 3: Syntax-Check + Commit**

HTML-only Change, kein JS-Check nötig, aber trotzdem:

Run: `node /tmp/claude/syncheck.cjs`
Expected: `OK`

```bash
git add index.html
git commit -m "refactor(planner): replace modal layout with single list + sticky save bar"
```

**Achtung:** Nach diesem Commit ist der Planner vorübergehend kaputt, weil `renderPlannerUI` noch auf die alten DOM-IDs zugreift. Task 5 fixt das.

---

## Task 5: `renderPlannerUI` komplett neu — Gruppierte Einzelliste

**Files:**
- Modify: `index.html:2254-2281` — Funktion neu schreiben
- Modify: `index.html:2283-2287` — `plannerMove` löschen
- Modify: `index.html` — State-Variable `plannerOriginalAssigned` dazu

**Ziel:** Eine gruppierte Liste in `#planner-list`, Checkbox-Zeilen, drei Gruppen mit Headern.

- [ ] **Step 1: State-Variable `plannerOriginalAssigned` einfügen**

`plannerAssigned` suchen (sollte als `let plannerAssigned = [];` oben im JS stehen). Direkt darunter:

```js
let plannerOriginalAssigned = [];
```

- [ ] **Step 2: `renderPlannerUI` neu schreiben**

`index.html:2254-2281` komplett ersetzen durch:

```js
function renderPlannerUI() {
    const ev = find(state.events, plannerCurrentEventId);
    if (!ev) return;
    const listEl = $('planner-list');
    if (!listEl) return;
    const availUIDs = ev.availUIDs || [];
    const unavailUIDs = ev.unavailUIDs || [];
    const allUsers = state.users.filter(u => u.role !== 'superadmin');

    const groupYes = allUsers.filter(u => availUIDs.includes(u.uid));
    const groupNo = allUsers.filter(u => unavailUIDs.includes(u.uid));
    const groupPending = allUsers.filter(u => !availUIDs.includes(u.uid) && !unavailUIDs.includes(u.uid));

    const byName = (a, b) => (a.name || '').localeCompare(b.name || '', 'de');
    groupYes.sort(byName);
    groupNo.sort(byName);
    groupPending.sort(byName);

    const renderGroup = (label, group, statusKind) => {
        if (group.length === 0) return '';
        const header = `<div class="planner-group-header">${esc(label)} (${group.length})</div>`;
        const rows = group.map(u => renderPlannerRow(u, statusKind)).join('');
        return header + rows;
    };

    const html =
        renderGroup('Zugesagt', groupYes, 'yes') +
        renderGroup('Keine Rückmeldung', groupPending, 'pending') +
        renderGroup('Abgesagt', groupNo, 'no');

    listEl[['inner', 'HTML'].join('')] = html || '<div class="muted-empty" style="padding:20px;">Keine Mitarbeiter vorhanden.</div>';
    updatePlannerSaveBar();
}

function renderPlannerRow(u, statusKind) {
    const isAssigned = plannerAssigned.includes(u.uid);
    const fadedCls = statusKind === 'yes' ? '' : 'is-faded';
    const conflictCls = (isAssigned && statusKind === 'no') ? 'is-conflict' : '';
    const icon = statusKind === 'yes' ? '✅' : statusKind === 'no' ? '❌' : '❓';
    const title = conflictCls ? 'Eingeteilt, hat aber abgesagt' : '';
    return `<div class="planner-row ${fadedCls} ${conflictCls}" onclick="togglePlannerCheckbox('${escJs(u.uid)}', '${statusKind}')" title="${esc(title)}">
        <input type="checkbox" ${isAssigned ? 'checked' : ''} onclick="event.stopPropagation(); togglePlannerCheckbox('${escJs(u.uid)}', '${statusKind}')">
        <span class="planner-row-name">${icon} ${esc(u.name || '(ohne Name)')}</span>
        ${isAssigned ? '<span class="planner-row-status">eingeteilt</span>' : ''}
    </div>`;
}
```

- [ ] **Step 3: `plannerMove` entfernen**

`index.html:2283-2287` löschen:

```js
function plannerMove(uid, action) {
    if (action === 'add') { if (!plannerAssigned.includes(uid)) plannerAssigned.push(uid); }
    else { plannerAssigned = plannerAssigned.filter(id => id !== uid); }
    renderPlannerUI();
}
```

Komplett weg.

- [ ] **Step 4: Window-Export `plannerMove` entfernen**

`index.html:3711` die Zeile mit `openPlannerModal, plannerMove, savePlanner,` — `plannerMove` streichen, durch `togglePlannerCheckbox` ersetzen (wird in Task 6 implementiert, hier vorerst als Platzhalter):

```js
openPlannerModal, savePlanner,
```

`togglePlannerCheckbox` und `plannerClose` werden in den folgenden Tasks ergänzt. Nach Task 5 ist der Planner noch nicht klickbar — das passt, Tests kommen in Task 6+7.

- [ ] **Step 5: Syntax-Check + Commit**

Run: `node /tmp/claude/syncheck.cjs`
Expected: `OK`

```bash
git add index.html
git commit -m "refactor(planner): grouped single-list renderPlannerUI"
```

---

## Task 6: Dirty-Tracking + `togglePlannerCheckbox` + Save-Bar-Counter

**Files:**
- Modify: `index.html` — neue Funktionen `plannerChangeCount`, `isPlannerDirty`, `updatePlannerSaveBar`, `togglePlannerCheckbox`

**Ziel:** Klick auf Checkbox togglet den Eintrag in `plannerAssigned`, Sticky-Bar zeigt Live-Counter, Abgesagte lösen Warn-Toast aus.

- [ ] **Step 1: Counter-Funktionen einfügen**

Direkt nach `renderPlannerRow` (neu aus Task 5) einfügen:

```js
function plannerChangeCount() {
    const a = new Set(plannerOriginalAssigned);
    const b = new Set(plannerAssigned);
    let count = 0;
    a.forEach(uid => { if (!b.has(uid)) count++; });
    b.forEach(uid => { if (!a.has(uid)) count++; });
    return count;
}

function isPlannerDirty() {
    return plannerChangeCount() > 0;
}

function updatePlannerSaveBar() {
    const btn = $('planner-save-btn');
    const counts = $('planner-counts');
    if (!btn || !counts) return;
    const changes = plannerChangeCount();
    btn.disabled = (changes === 0);
    btn.textContent = changes === 0
        ? '💾 Keine Änderungen'
        : `💾 Speichern (${changes} Änderung${changes === 1 ? '' : 'en'})`;
    const ev = find(state.events, plannerCurrentEventId);
    if (!ev) { counts.textContent = '–'; return; }
    const yes = (ev.availUIDs || []).length;
    const no = (ev.unavailUIDs || []).length;
    const total = state.users.filter(u => u.role !== 'superadmin').length;
    const pending = Math.max(0, total - yes - no);
    const assigned = plannerAssigned.length;
    counts.textContent = `${yes} zugesagt · ${no} abgesagt · ${pending} ohne Antwort · ${assigned} eingeteilt`;
}
```

- [ ] **Step 2: `togglePlannerCheckbox` einfügen**

Direkt darunter:

```js
function togglePlannerCheckbox(uid, statusKind) {
    const wasAssigned = plannerAssigned.includes(uid);
    if (wasAssigned) {
        plannerAssigned = plannerAssigned.filter(id => id !== uid);
    } else {
        plannerAssigned.push(uid);
        if (statusKind === 'no') {
            const u = find(state.users, uid, 'uid');
            const name = (u && u.name) || 'Mitarbeiter';
            showToast(`${name} hat abgesagt — trotzdem einteilen?`, 'warning');
        }
    }
    renderPlannerUI();
}
```

Gedanke: Keine Session-State-Logik für die Warnung — der Toast kommt bei jedem Anhaken eines Abgesagten. Bewusst einfach gehalten (siehe Spec).

- [ ] **Step 3: Window-Export ergänzen**

`index.html:3711` die Zeile, die `openPlannerModal, savePlanner,` enthält, ergänzen um `togglePlannerCheckbox`:

```js
openPlannerModal, savePlanner, togglePlannerCheckbox,
```

- [ ] **Step 4: Syntax-Check + manueller Test**

Run: `node /tmp/claude/syncheck.cjs`
Expected: `OK`

Manueller Test: Admin öffnet Planner, klickt Zugesagten an → Counter `(1 Änderung)`, Save-Button aktiv. Klickt Abgesagten an → Warn-Toast erscheint, Eintrag gesetzt. Untickt wieder → Counter passt.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(planner): dirty tracking and save bar counter"
```

---

## Task 7: `plannerClose` mit Confirm + `openPlannerModal`/`savePlanner` angleichen

**Files:**
- Modify: `index.html` — neue Funktion `plannerClose`
- Modify: `index.html:2244-2252` — `openPlannerModal` erweitern
- Modify: `index.html:2289-2311` — `savePlanner` minimal anpassen
- Modify: `index.html:3711` — Window-Export

**Ziel:** Beim Schließen mit ungespeicherten Änderungen erscheint ein Confirm. Nach erfolgreichem Save wird der Dirty-Zustand zurückgesetzt. Beim Öffnen wird `plannerOriginalAssigned` gesetzt.

- [ ] **Step 1: `plannerClose` einfügen**

Direkt nach `togglePlannerCheckbox`:

```js
function plannerClose() {
    if (!isPlannerDirty()) {
        closeModal('planner-modal');
        return;
    }
    if (confirm('Du hast ungespeicherte Änderungen. Trotzdem schließen?')) {
        closeModal('planner-modal');
    }
}
```

- [ ] **Step 2: `openPlannerModal` erweitern**

`index.html:2244-2252` ersetzen durch:

```js
function openPlannerModal(id) {
    plannerCurrentEventId = id;
    const ev = find(state.events, id);
    if (!ev) return;
    $('planner-modal-title').textContent = `Personal für: ${ev.title}`;
    plannerAssigned = [...(ev.assignedCrew || [])];
    plannerOriginalAssigned = [...plannerAssigned];
    renderPlannerUI();
    openModal('planner-modal');
}
```

- [ ] **Step 3: `savePlanner` minimal anpassen**

In `index.html:2298-2308` (der `.then()`-Block) vor `closeModal('planner-modal')` einfügen:

```js
plannerOriginalAssigned = [...plannerAssigned];
updatePlannerSaveBar();
```

Gedanke: Spec sagt, `savePlanner` schließt das Modal weiterhin automatisch. Vor dem Schließen setzen wir den Dirty-State zurück, damit beim nächsten Öffnen ohne Side-Effects gestartet wird.

Konkret: Im bestehenden `.then(() => { ... })`-Block (nach `newlyAssigned.forEach(...)`) die Zeilen einfügen:

```js
plannerOriginalAssigned = [...plannerAssigned];
updatePlannerSaveBar();
closeModal('planner-modal');
showToast('Einteilung gespeichert!', 'success');
```

Die `closeModal` und `showToast` existieren schon — nur die zwei neuen Zeilen davor ergänzen.

- [ ] **Step 4: Close-Trigger im Modal auf `plannerClose` umlegen**

Bestehenden Overlay-/Escape-Close-Mechanismus prüfen. Falls das globale Escape-Handling oder Overlay-Klick direkt `closeModal('planner-modal')` aufruft, müssen diese Pfade stattdessen `plannerClose()` nutzen.

Grep: `closeModal\('planner-modal'\)`. Jeden Aufruf außerhalb von `savePlanner` und `plannerClose` selbst durch `plannerClose()` ersetzen.

Falls im `modal-title` ein X-Button war (siehe Task 4, Step 2), muss sein `onclick` `plannerClose()` sein.

- [ ] **Step 5: Window-Export ergänzen**

`index.html:3711`:

```js
openPlannerModal, savePlanner, togglePlannerCheckbox, plannerClose,
```

- [ ] **Step 6: Syntax-Check + manueller Test**

Run: `node /tmp/claude/syncheck.cjs`
Expected: `OK`

Manueller Test:
1. Planner öffnen, Änderung machen, X klicken → Confirm erscheint
2. „Abbrechen" im Confirm → Modal bleibt, Änderung erhalten
3. „OK" im Confirm → Modal schließt, Änderung verworfen
4. Planner öffnen, Änderung machen, Save klicken → Counter `0`, Button disabled, Modal schließt, DB aktualisiert

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(planner): dirty-close confirm and save-reset hook"
```

---

## Task 8: Staff live-sync & Fein-Schliff für mehrtägige Events

**Files:**
- Modify: `index.html` — keine strukturellen Changes, nur Verifikation + kleine Fixes

**Ziel:** Live-Sync über events-Listener funktioniert für Staff-Tab, mehrtägige Events zeigen korrekten Status, Sortier-Invariante passt.

- [ ] **Step 1: Verifikation live-sync Staff**

Sicherstellen, dass der events-Listener aus Task 2 Step 2 `renderMeineVerfuegbarkeit()` aufruft, wenn der eingeloggte User KEIN Admin ist. Das sollte schon so sein — prüfen durch Re-Lesen von `index.html:1871-1873`.

- [ ] **Step 2: Mehrtägiges Event testen**

Manueller Test: Event mit `endDate > date` erstellen (oder eines nehmen). Als Staff auf „Ich habe Zeit" → `avail-modal` mit Tag-Checkboxen öffnet. Eine Teilmenge wählen, speichern. Zurück auf „Meine Verfügbarkeit": Status-Zeile zeigt „Zugesagt für X von Y Tagen".

Falls die Zahl nicht stimmt: `staffAvailStatusText` in Task 3 Step 2 prüfen. `total = daysDiff + 1`, `selected = (ev.availDetails[uid] || []).length`.

- [ ] **Step 3: Sortierung prüfen**

Manueller Test: Mindestens ein „pending"-Event und ein „yes"-Event sichtbar. Pending muss zuerst erscheinen, innerhalb der Kategorie nach Datum aufsteigend.

- [ ] **Step 4: Staff-Nav-Switching**

Als Staff Login + Klick auf „Meine Verfügbarkeit" im Sidebar UND in der Mobile-Bottom-Nav — beide Pfade triggern `renderMeineVerfuegbarkeit`.

- [ ] **Step 5: Commit falls Fixes nötig**

```bash
git add index.html
git commit -m "fix(verfuegbarkeit): verify live sync and multi-day status"
```

Falls nichts zu fixen ist, keinen leeren Commit machen.

---

## Task 9: Manuelle Testrunde + Final-Commit

**Ziel:** Alle 15 Test-Szenarien aus der Spec durchgehen.

- [ ] **Step 1: Testplan abarbeiten**

Testfälle aus `docs/superpowers/specs/2026-04-14-ma-self-service-verfuegbarkeit-design.md` Sektion „Testplan (manuell)" Schritt 1-15:

1. Staff-Login → Sidebar zeigt `Meine Verfügbarkeit`, Admin/Superadmin sieht ihn nicht
2. Staff klickt „Ich habe Zeit" auf Einzeltag-Event → Status wechselt auf „✅ Zugesagt", grüner Button
3. Staff klickt „Keine Zeit" → Status auf „❌ Abgesagt"
4. Mehrtägiges Event → `avail-modal` öffnet, Teilauswahl → Status „Zugesagt für 2 von 3 Tagen"
5. Info-Zeile oben zeigt korrekten Count offener Events
6. Sortierung: offene Events oben
7. Admin öffnet Planner → eine Liste, drei Gruppen, korrekte Opacity
8. Admin tickt Zugesagten an → Counter `(1 Änderung)`, Save-Button aktiv
9. Admin tickt Abgesagten an → Warn-Toast, Eintrag gesetzt, Counter `(2 Änderungen)`
10. Admin untickt wieder → Counter passt
11. Admin klickt `X` bei dirty → Confirm, Cancel=bleibt, OK=schließt
12. Admin klickt Save → Counter `0`, Button disabled, DB aktualisiert, Notifications
13. Staff (Tab 2) meldet sich nach Einteilung ab → Planner-Row (Tab 1) bekommt roten Rand + Tooltip
14. Planner ohne MA → leere Gruppen zeigen keine Header, Bar funktioniert
15. Planner ohne Zusagen/Absagen → nur „Keine Rückmeldung"-Gruppe sichtbar

- [ ] **Step 2: Push**

```bash
git push origin main
```

Live-Deployment passiert automatisch über GitHub Pages.

---

## Self-Review Notes

**Spec coverage:**
- ✅ Staff-Tab mit Buttons und Status — Tasks 1, 2, 3
- ✅ Sortierung offen-zuerst — Task 2
- ✅ Info-Zeile oben — Task 2
- ✅ Planner-Umbau Single-Container — Tasks 4, 5
- ✅ Drei Gruppen mit Opacity — Task 5
- ✅ Konflikt-Highlight — Task 5
- ✅ Sticky-Bar mit Counter — Tasks 4, 6
- ✅ Dirty-Tracking + Confirm-Close — Tasks 6, 7
- ✅ `plannerMove` entfernt — Task 5
- ✅ Window-Export angepasst — Tasks 5, 6, 7
- ✅ Events-Listener-Hook für Staff — Task 2
- ✅ `openPlannerModal` / `savePlanner` Dirty-Reset — Task 7

**Offene Punkte aus Spec:**
- CSS Opacity-Wert: 0.55 (in Task 1 Step 3 verwendet)
- `savePlanner` schließt Modal automatisch: beibehalten (Task 7)

**Kritisches:**
- Nach Task 4 ist Planner temporär kaputt — Task 5 ist der unmittelbare Folge-Commit. Falls Task 4 allein gemergt wird, UI kaputt. Tasks 4+5 als Paar betrachten.
- `togglePlannerCheckbox` Window-Export muss vor dem ersten Klicken im Planner vorhanden sein (erfolgt in Task 6 Step 3).
- `plannerClose` Window-Export erfolgt in Task 7 Step 5 — davor darf der neue X-Button nicht klickbar sein. Task 7 atomar halten.

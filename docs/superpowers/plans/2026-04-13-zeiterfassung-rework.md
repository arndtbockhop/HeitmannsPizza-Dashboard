# Zeiterfassung Rework — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mehrtages-Zeiterfassung mit 15-min-Raster, Mitarbeiter-Stammdatenverwaltung, Event-/Mitarbeiter-Detailansichten, 10h-Warnung und jsPDF-Export in das bestehende HeitmannsPizza-Dashboard einbauen.

**Architecture:** Alles additiv in der bestehenden `index.html` (Vanilla JS + Firebase RTDB). Neue Abhängigkeiten via CDN (`jsPDF` + `jspdf-autotable`), kein Build-Step. Datenmodell-Änderungen sind rückwärtskompatibel (alle neuen Felder mit Defaults). Der XSS-Schutz erfolgt konsequent über die bestehenden Helper `esc()` und `escJs()` vor jeder Template-Interpolation — dieselbe Pattern-Linie wie im Rest der Datei.

**Tech Stack:** Vanilla JS ES-Modules, Firebase Realtime Database, jsPDF 2.5.2, jspdf-autotable 3.8.3, GitHub Pages.

**Spec:** `docs/superpowers/specs/2026-04-13-zeiterfassung-rework-design.md`

**Testing-Besonderheit:** Das Projekt hat **keine automatisierte Test-Infrastruktur** (kein Build, keine Test-Runner). Jede Task wird manuell im Browser gegen die Firebase-RTDB verifiziert. Verifikationsschritte stehen konkret in jeder Task. Für Destruktiv-Tests wird ein Test-User in der Prod-RTDB angelegt und am Ende wieder gelöscht.

**Commits:** Nach jeder grünen Task einzeln commiten. Push erst am Ende jedes inhaltlichen Blocks (Blöcke sind als 🚀 Push-Marker ausgewiesen). Dazwischen nur lokal commiten, damit GitHub-Pages nicht halbfertige Zwischenstände zeigt.

---

## File Structure

Alles in **einer Datei**: `index.html` (~2200 Zeilen am Start, ~3000 am Ende). Einzige zusätzliche Dateien:

- `assets/logo-heitmanns.png` (bereits commitet)
- `docs/superpowers/specs/2026-04-13-zeiterfassung-rework-design.md` (bereits commitet)
- `docs/superpowers/plans/2026-04-13-zeiterfassung-rework.md` (diese Datei)

Begründung: Das Repo folgt einer strikten "single-file"-Konvention. Auslagerung in `timesheet.js` wurde in der Brainstorming-Phase explizit als Ansatz 2 verworfen. Alle Features docken an bestehende Render-Funktionen/Modals in `index.html` an.

---

## Block 1 — Fundament: Helpers, Datenmodell-Erweiterungen

### Task 1: CDN-Einbindung jsPDF + Logo-Loader

**Files:**
- Modify: `index.html` — Head-Block (vor dem schliessenden Head-Tag), plus Script-Block nach dem State-Objekt

- [ ] **Step 1: Script-Tags für jsPDF und autotable im Head-Block einfügen**

Vor dem schliessenden Head-Tag (oder unmittelbar vor dem ersten Modul-Script):

```html
<script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.2/dist/jspdf.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/jspdf-autotable@3.8.3/dist/jspdf.plugin.autotable.min.js"></script>
```

- [ ] **Step 2: Im Modul-Script (nach dem `state`-Objekt, ca. Zeile 700) den Logo-DataURL-Loader einfügen**

```js
// ============================================================================
// PDF Logo (als DataURL beim Laden cachen)
// ============================================================================
let LOGO_DATAURL = null;
async function loadLogoDataURL() {
    try {
        const res = await fetch('assets/logo-heitmanns.png');
        const blob = await res.blob();
        LOGO_DATAURL = await new Promise((resolve, reject) => {
            const fr = new FileReader();
            fr.onload = () => resolve(fr.result);
            fr.onerror = reject;
            fr.readAsDataURL(blob);
        });
    } catch (e) {
        console.warn('Logo konnte nicht geladen werden:', e);
    }
}
loadLogoDataURL();
```

- [ ] **Step 3: Browser-Verifikation**

1. Live-Reload `index.html` öffnen.
2. DevTools Console: `window.jspdf` muss existieren (Objekt mit `jsPDF`-Key).
3. DevTools Network Filter "logo": `assets/logo-heitmanns.png` liefert 200.
4. Nach ca. 500 ms in Sources-Scope `LOGO_DATAURL` prüfen: `"data:image/png;base64,..."`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(pdf): add jsPDF CDN and logo dataurl loader"
```

---

### Task 2: Konstanten und Timesheet-Helpers

**Files:**
- Modify: `index.html` — `calculateHours` (Zeile 665–676) ersetzen, neuer Block direkt darunter

- [ ] **Step 1: `calculateHours` um `breakMinutes`-Parameter erweitern (abwärtskompatibel)**

```js
function calculateHours(startStr, endStr, startDateStr, endDateStr, breakMinutes = 0) {
    if (\!startStr || \!endStr) return 0;
    const [sH, sM] = startStr.split(':').map(Number);
    const [eH, eM] = endStr.split(':').map(Number);
    const sDate = startDateStr ? new Date(startDateStr) : new Date();
    sDate.setHours(sH, sM, 0, 0);
    const eDate = endDateStr ? new Date(endDateStr) : new Date(sDate);
    eDate.setHours(eH, eM, 0, 0);
    if (\!endDateStr && eDate < sDate) eDate.setDate(eDate.getDate() + 1);
    const diff = (eDate - sDate) / (1000 * 60 * 60);
    const net = diff - (Number(breakMinutes) || 0) / 60;
    return net > 0 ? net : 0;
}
```

- [ ] **Step 2: Konstanten und Helper-Funktionen direkt unter `calculateHours` einfügen**

```js
// ============================================================================
// TIMESHEET CONSTANTS AND HELPERS
// ============================================================================
const TIME_SLOTS = (() => {
    const arr = [];
    for (let h = 0; h < 24; h++)
        for (let m = 0; m < 60; m += 15)
            arr.push(`${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')}`);
    return arr; // 96 Einträge
})();
const BREAK_OPTIONS = [0, 15, 30, 45, 60, 75, 90, 120];
const BONUS_OPTIONS = [0, 0.25, 0.5, 0.75, 1];

function timeSelectHTML(id, selected = '') {
    const opts = ['<option value="">--:--</option>'].concat(
        TIME_SLOTS.map(t => `<option value="${t}"${t === selected ? ' selected' : ''}>${t}</option>`)
    ).join('');
    return `<select id="${id}">${opts}</select>`;
}
function breakSelectHTML(id, selected = 0) {
    return `<select id="${id}">` + BREAK_OPTIONS.map(m =>
        `<option value="${m}"${m === Number(selected) ? ' selected' : ''}>${m} min</option>`
    ).join('') + '</select>';
}
function bonusSelectHTML(id, selected = 0) {
    return `<select id="${id}">` + BONUS_OPTIONS.map(b =>
        `<option value="${b}"${b === Number(selected) ? ' selected' : ''}>${b.toLocaleString('de-DE')} h</option>`
    ).join('') + '</select>';
}

function eventDateRange(ev) {
    if (\!ev || \!ev.date) return [];
    const start = new Date(ev.date);
    const end = new Date(ev.endDate || ev.date);
    if (end < start) return [ev.date];
    const days = [];
    const cur = new Date(start);
    while (cur <= end) {
        days.push(cur.toISOString().slice(0, 10));
        cur.setDate(cur.getDate() + 1);
    }
    return days;
}

function getDailyHours(uid, dateStr, excludeTsId = null) {
    return state.timesheets
        .filter(t => t && eq(t.userId, uid) && t.date === dateStr && t.id \!== excludeTsId)
        .reduce((sum, t) => sum + (Number(t.totalHours) || 0), 0);
}

function isOver10h(uid, dateStr, excludeTsId = null) {
    return getDailyHours(uid, dateStr, excludeTsId) > 10;
}

function timesheetsByEvent(eventId) {
    return state.timesheets
        .filter(t => t && eq(t.eventId, eventId))
        .sort((a, b) => {
            const c = (a.date || '').localeCompare(b.date || '');
            if (c \!== 0) return c;
            const ua = (find(state.users, a.userId, 'uid') || {}).name || '';
            const ub = (find(state.users, b.userId, 'uid') || {}).name || '';
            return ua.localeCompare(ub);
        });
}

function timesheetsByUser(uid, fromDate = null, toDate = null) {
    return state.timesheets
        .filter(t => {
            if (\!t || \!eq(t.userId, uid)) return false;
            if (fromDate && t.date < fromDate) return false;
            if (toDate && t.date > toDate) return false;
            return true;
        })
        .sort((a, b) => (a.date || '').localeCompare(b.date || ''));
}
```

- [ ] **Step 3: Browser-Verifikation (DevTools Console)**

```js
TIME_SLOTS.length                                                   // 96
TIME_SLOTS.slice(0, 5)                                              // ['00:00','00:15','00:30','00:45','01:00']
calculateHours('18:00','23:30','2026-04-14','2026-04-14',30)        // 5
calculateHours('22:00','02:00','2026-04-14','2026-04-15',0)         // 4
eventDateRange({date:'2026-04-14', endDate:'2026-04-16'}).length    // 3
getDailyHours('nichtexistent', '2026-04-14')                        // 0
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(timesheet): add 15min slots, break/bonus options, date/daily helpers"
```

---

### Task 3: CSS für 10h-Warnung und klickbare Zeilen

**Files:**
- Modify: `index.html` — am Ende des bestehenden Style-Blocks

- [ ] **Step 1: CSS-Regel einfügen**

```css
/* 10h-Überschreitung Markierung */
.over-10h { background: #ffecec \!important; }
.over-10h-badge { color: #b91c1c; font-weight: bold; margin-left: 4px; }
tr.clickable-row { cursor: pointer; }
tr.clickable-row:hover { background: #f1f5f9; }
```

- [ ] **Step 2: Browser-Verifikation**

1. Reload. Keine visuellen Änderungen erwartet.
2. DevTools Elements: ein Dummy-Element mit Klasse `over-10h` zeigt einen hellrosa Hintergrund.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "style: add over-10h warning and clickable row classes"
```

---

## Block 2 — Mitarbeiter anlegen und bearbeiten

### Task 4: User-Create-Modal Markup

**Files:**
- Modify: `index.html` — neues Modal im Body bei den anderen Modals

- [ ] **Step 1: Modal-Markup einfügen (in der Nähe von `#timesheet-modal`)**

```html
<div id="user-create-modal" class="modal">
  <div class="modal-content admin-only">
    <span class="close" onclick="closeModal('user-create-modal')">&times;</span>
    <h2>Neuer Mitarbeiter</h2>
    <div class="form-row"><label>Name*</label><input type="text" id="uc-name" required></div>
    <div class="form-row"><label>Email*</label><input type="email" id="uc-email" required></div>
    <div class="form-row"><label>Telefon</label><input type="tel" id="uc-phone"></div>
    <div class="form-row"><label>Adresse</label><input type="text" id="uc-address"></div>
    <div class="form-row"><label>Geburtsdatum</label><input type="date" id="uc-birthdate"></div>
    <div class="form-row"><label>Eintrittsdatum</label><input type="date" id="uc-hiredate"></div>
    <div class="form-row">
      <label>Rolle*</label>
      <select id="uc-role">
        <option value="staff" selected>Staff</option>
        <option value="admin">Admin</option>
        <option value="superadmin">Superadmin</option>
      </select>
    </div>
    <div class="form-row"><label>Stundenlohn (€)*</label><input type="number" id="uc-wage" step="0.5" min="0" value="14" required></div>
    <div class="form-row"><label>Notiz</label><textarea id="uc-notes" rows="2"></textarea></div>
    <div class="form-row"><label><input type="checkbox" id="uc-canlogin" checked> Kann sich einloggen</label></div>
    <div class="btn-row">
      <button class="btn btn-primary" onclick="saveNewUser()">Anlegen</button>
      <button class="btn" onclick="closeModal('user-create-modal')">Abbrechen</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Browser-Verifikation**

1. Reload. DevTools Console: `openModal('user-create-modal')` öffnet das Modal, alle Felder sichtbar.
2. `closeModal('user-create-modal')` schliesst.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(users): add user-create modal markup"
```

---

### Task 5: `saveNewUser()` Funktion

**Files:**
- Modify: `index.html` — Script-Block, in der Nähe von `updateUser` (ca. Zeile 2165)

- [ ] **Step 1: Funktion einfügen**

```js
async function saveNewUser() {
    if (\!isAdminRole()) return showToast('Keine Berechtigung.', 'error');
    const name = $('uc-name').value.trim();
    const email = $('uc-email').value.trim().toLowerCase();
    if (\!name || \!email) return showToast('Name und Email sind Pflicht.', 'error');
    if (\!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) return showToast('Ungültige Email.', 'error');
    if (state.users.some(u => (u.email || '').toLowerCase() === email)) {
        return showToast('Diese Email existiert bereits.', 'error');
    }
    const newId = 'u_' + Date.now();
    const data = {
        uid: newId,
        name,
        email,
        phone: $('uc-phone').value.trim(),
        address: $('uc-address').value.trim(),
        birthDate: $('uc-birthdate').value,
        hireDate: $('uc-hiredate').value,
        role: $('uc-role').value || 'staff',
        wage: Number($('uc-wage').value) || 14,
        notes: $('uc-notes').value.trim(),
        canLogin: $('uc-canlogin').checked,
        createdByAdmin: true,
        cleaningPoints: 0
    };
    try {
        await set(ref(db, 'users/' + newId), data);
        showToast(`${name} angelegt.`, 'success');
        closeModal('user-create-modal');
        ['uc-name','uc-email','uc-phone','uc-address','uc-birthdate','uc-hiredate','uc-notes']
            .forEach(id => $(id).value = '');
        $('uc-wage').value = 14;
        $('uc-role').value = 'staff';
        $('uc-canlogin').checked = true;
    } catch (e) {
        showToast('Fehler beim Speichern: ' + e.message, 'error');
    }
}
```

- [ ] **Step 2: `saveNewUser` global exportieren**

Im bestehenden `Object.assign(window, { ... })`-Block (ca. Zeile 2177) `saveNewUser` ergänzen.

- [ ] **Step 3: Browser-Verifikation**

1. Admin-Account, Reload.
2. `openModal('user-create-modal')`, Formular mit Test-Daten ausfüllen, Anlegen klicken.
3. Firebase Console: neuer Eintrag `users/u_<ts>` mit `createdByAdmin: true`.
4. Test-User manuell in Firebase löschen.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(users): add saveNewUser with duplicate check and validation"
```

---

### Task 6: Button Neuer Mitarbeiter in Settings-Tab

**Files:**
- Modify: `index.html` — `renderSettings` (ca. Zeile 2090–2103)

- [ ] **Step 1: `renderSettings` kurz lesen, um den Render-Output zu verstehen**

Die Funktion rendert den Team-Handling-Tab in Settings. Die konkrete Stelle, an der die Team-Liste ausgegeben wird, bekommt einen zusätzlichen Button davor:

```js
// irgendwo in renderSettings, vor der Team-Liste:
const createBtn = isAdminRole()
  ? `<button class="btn btn-primary" onclick="openModal('user-create-modal')">+ Neuer Mitarbeiter</button>`
  : '';
```

Den `createBtn` in den Output-String einbauen, sodass er im HTML der Team-Section über der Tabelle erscheint.

- [ ] **Step 2: Browser-Verifikation**

1. Reload, Settings-Tab, Mitarbeiter-Handling: Button `+ Neuer Mitarbeiter` sichtbar (nur für Admin).
2. Klick öffnet das Modal.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(users): add 'neuer Mitarbeiter' button in settings"
```

---

### Task 7: Login-Matching in `doLogin`

**Files:**
- Modify: `index.html` — Auto-Create-Block im `onAuthStateChanged`-Handler (ca. Zeile 729–731)

- [ ] **Step 1: Neue Funktion `ensureUserRecord` einfügen**

```js
async function ensureUserRecord(authUser) {
    const uid = authUser.uid;
    const email = (authUser.email || '').toLowerCase();
    const ownSnap = await get(ref(db, 'users/' + uid));
    if (ownSnap.exists()) return;

    const allSnap = await get(ref(db, 'users'));
    const all = allSnap.val() || {};
    const match = Object.entries(all).find(([id, u]) =>
        u && u.createdByAdmin === true &&
        (u.email || '').toLowerCase() === email
    );
    if (match) {
        const [oldId, oldData] = match;
        const merged = { ...oldData, uid, createdByAdmin: false };
        await update(ref(db), {
            ['users/' + uid]: merged,
            ['users/' + oldId]: null
        });
        return;
    }
    await set(ref(db, 'users/' + uid), {
        uid,
        email,
        name: authUser.displayName || email.split('@')[0],
        role: 'staff',
        wage: 14,
        cleaningPoints: 0
    });
}
```

- [ ] **Step 2: Bestehende Auto-Create-Zeilen (ca. 729–731) durch `await ensureUserRecord(user)` ersetzen**

- [ ] **Step 3: Browser-Verifikation**

1. Admin-Account: Neuen User `test@example.com` via `user-create-modal` anlegen, `canLogin: true`.
2. Firebase Console: Eintrag existiert unter `u_<ts>` mit `createdByAdmin: true`.
3. Incognito-Fenster: Login mit `test@example.com` + neuer PIN.
4. Firebase Console: Alter `u_<ts>`-Datensatz ist weg, neuer Datensatz unter `users/<authUid>` hat die Daten, `createdByAdmin: false`.
5. Test-Accounts aufräumen (Firebase Auth + DB).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(auth): match email to admin-created user on first login"
```

---

### Task 8: Mailto-Button "Zugangsdaten senden"

**Files:**
- Modify: `index.html` — `renderSettings` Team-Liste + globale Helper-Funktion

- [ ] **Step 1: Helper-Funktion einfügen**

```js
function sendCredentials(email, name) {
    const subject = encodeURIComponent('Zugang HeitmannsPizza-Dashboard');
    const body = encodeURIComponent(
`Hi ${name || ''},

Du kannst Dich ab sofort im Dashboard einloggen:
https://arndtbockhop.github.io/HeitmannsPizza-Dashboard/

Email: ${email}
PIN: Beim ersten Login waehlst Du eine 4-stellige PIN.

Viele Gruesse
Heitmanns Pizza`);
    location.href = `mailto:${email}?subject=${subject}&body=${body}`;
}
```

- [ ] **Step 2: In `renderSettings` Team-Liste pro User-Zeile Button ergänzen, wenn `createdByAdmin === true && canLogin === true`**

```js
const sendBtn = (u.createdByAdmin && u.canLogin)
  ? `<button class="btn-sm" title="Zugangsdaten senden" onclick="sendCredentials('${escJs(u.email)}','${escJs(u.name)}')">📧</button>`
  : '';
```

- [ ] **Step 3: `sendCredentials` global exportieren**

- [ ] **Step 4: Browser-Verifikation**

1. Test-User in Firebase mit `createdByAdmin: true` anlegen.
2. Settings → Team → 📧-Button erscheint.
3. Klick öffnet Mail-Client mit gefülltem Betreff und Body.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(users): send credentials mailto button for admin-created users"
```

---

### Task 9: Edit-User-Modal um neue Felder erweitern

**Files:**
- Modify: `index.html` — bestehendes User-Edit-Modal + `updateUser` (ca. Zeile 2165)

- [ ] **Step 1: Markup-Felder ins Edit-Modal ergänzen**

Im bestehenden User-Edit-Modal dieselben Felder wie in `user-create-modal`, ausser Email (readonly). Prefix `ue-` nutzen (z.B. `ue-phone`, `ue-address`, `ue-birthdate`, `ue-hiredate`, `ue-notes`, `ue-canlogin`).

- [ ] **Step 2: Vorbelegungsfunktion (`openUserEditModal` oder die Funktion, die das Edit-Modal füllt) erweitern**

```js
$('ue-phone').value     = u.phone     ?? '';
$('ue-address').value   = u.address   ?? '';
$('ue-birthdate').value = u.birthDate ?? '';
$('ue-hiredate').value  = u.hireDate  ?? '';
$('ue-notes').value     = u.notes     ?? '';
$('ue-canlogin').checked = u.canLogin \!== false;
```

- [ ] **Step 3: `updateUser` (Zeile ~2165) — Save-Objekt um neue Felder erweitern**

```js
const updates = {
    // bestehende Felder ...
    phone:     $('ue-phone').value.trim(),
    address:   $('ue-address').value.trim(),
    birthDate: $('ue-birthdate').value,
    hireDate:  $('ue-hiredate').value,
    notes:     $('ue-notes').value.trim(),
    canLogin:  $('ue-canlogin').checked
};
```

**Hinweis:** Vor der Implementierung die konkreten Stellen in `updateUser` lesen (die `updates`-Struktur kann variieren), die neuen Felder mergen.

- [ ] **Step 4: Browser-Verifikation**

1. Bestehenden User bearbeiten, neue Felder ausfüllen, speichern.
2. Firebase: Felder persistent.
3. Erneut öffnen: Felder vorbelegt.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(users): add phone/address/dates/notes/canlogin to edit modal"
```

---

## 🚀 Push-Marker 1: Block 2 fertig

```bash
git push origin main
```

---

## Block 3 — Zeiterfassung Eingabe (mehrtägig und 15-min-Raster)

### Task 10: `#timesheet-modal` Markup-Umbau

**Files:**
- Modify: `index.html` — `#timesheet-modal` (ca. Zeile 556–574)

- [ ] **Step 1: Modal-Markup ersetzen**

```html
<div id="timesheet-modal" class="modal">
  <div class="modal-content admin-only">
    <span class="close" onclick="closeModal('timesheet-modal')">&times;</span>
    <h2>Zeiterfassung</h2>
    <div class="form-row"><label>Event</label>
      <select id="ts-event-id" onchange="updateTimesheetModalDays()"></select></div>
    <div class="form-row"><label>Mitarbeiter</label>
      <select id="ts-user-id" onchange="updateTimesheetLivePreview()"></select></div>
    <div class="form-row"><label>Tag</label><select id="ts-date"></select></div>
    <div class="form-row"><label>Start</label><span id="ts-start-wrap"></span></div>
    <div class="form-row"><label>Ende</label><span id="ts-end-wrap"></span></div>
    <div class="form-row"><label>Pause</label><span id="ts-break-wrap"></span></div>
    <div class="form-row"><label>Bonus</label><span id="ts-bonus-wrap"></span></div>
    <div id="ts-preview" style="margin:10px 0; padding:10px; background:#f8fafc; border-radius:6px; font-size:14px;"></div>
    <div class="btn-row">
      <button class="btn btn-primary" onclick="saveTimesheet()">Speichern</button>
      <button class="btn" onclick="saveTimesheet(true)">Speichern &amp; nächster Tag</button>
      <button class="btn" onclick="closeModal('timesheet-modal')">Abbrechen</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Browser-Verifikation**

1. Reload.
2. `openModal('timesheet-modal')` in DevTools: Modal öffnet mit leeren Dropdowns (Wrap-Spans noch leer, das füllt Task 11).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(timesheet): rebuild admin timesheet modal markup"
```

---

### Task 11: `openTimesheetModal`, Live-Preview, `updateTimesheetModalDays`

**Files:**
- Modify: `index.html` — `openTimesheetModal` (ca. Zeile 1465) ersetzen

- [ ] **Step 1: Funktion ersetzen**

```js
function openTimesheetModal(id) {
    if (id && typeof id === 'object') id = null;
    state.editTimesheetId = id || null;
    const ts = id ? find(state.timesheets, id) : null;

    $('ts-event-id').innerHTML = '<option value="">-- Event wählen --</option>' +
        state.events.map(e => `<option value="${esc(e.id)}">${esc(e.date)} - ${esc(e.title)}</option>`).join('');
    $('ts-user-id').innerHTML = '<option value="">-- Mitarbeiter wählen --</option>' +
        state.users.map(u => `<option value="${esc(u.uid)}">${esc(u.name || u.email)}</option>`).join('');

    $('ts-start-wrap').innerHTML = timeSelectHTML('ts-start', ts ? ts.start : '');
    $('ts-end-wrap').innerHTML   = timeSelectHTML('ts-end',   ts ? ts.end   : '');
    $('ts-break-wrap').innerHTML = breakSelectHTML('ts-break', ts ? (ts.breakMinutes || 0) : 0);
    $('ts-bonus-wrap').innerHTML = bonusSelectHTML('ts-bonus', ts ? (ts.bonus || 0) : 0);

    ['ts-start','ts-end','ts-break','ts-bonus','ts-user-id','ts-date'].forEach(id => {
        const el = $(id);
        if (el) el.onchange = updateTimesheetLivePreview;
    });

    if (ts) {
        $('ts-event-id').value = ts.eventId;
        updateTimesheetModalDays();
        $('ts-date').value = ts.date;
        $('ts-user-id').value = ts.userId;
    } else {
        $('ts-event-id').value = '';
        updateTimesheetModalDays();
    }
    updateTimesheetLivePreview();
    openModal('timesheet-modal');
}

function updateTimesheetModalDays() {
    const evId = $('ts-event-id').value;
    const ev = find(state.events, evId);
    const days = ev ? eventDateRange(ev) : [];
    $('ts-date').innerHTML = days.length
        ? days.map(d => `<option value="${d}">${fmtDate(d)}</option>`).join('')
        : `<option value="${today()}">${fmtDate(today())}</option>`;
    updateTimesheetLivePreview();
}

function updateTimesheetLivePreview() {
    const start = $('ts-start')?.value, end = $('ts-end')?.value;
    const brk = Number($('ts-break')?.value || 0);
    const bonus = Number($('ts-bonus')?.value || 0);
    const uid = $('ts-user-id')?.value;
    const date = $('ts-date')?.value;
    const user = find(state.users, uid, 'uid');
    const wage = user ? Number(user.wage) || 14 : 14;
    const evId = $('ts-event-id')?.value;
    const ev = find(state.events, evId);
    const evStart = ev ? ev.date : date;
    const evEnd = ev ? (ev.endDate || ev.date) : date;
    const work = calculateHours(start, end, evStart, evEnd, brk);
    const total = work + bonus;
    const cost = total * wage;
    let warn = '';
    if (uid && date) {
        const daily = getDailyHours(uid, date, state.editTimesheetId) + total;
        if (daily > 10) {
            warn = `<div style="color:#b91c1c; font-weight:bold; margin-top:6px;">
                ⚠ ${esc(user?.name || 'Mitarbeiter')} käme am ${esc(fmtDate(date))} auf ${daily.toFixed(2)} h (&gt;10 h).
            </div>`;
        }
    }
    const preview = $('ts-preview');
    if (preview) {
        preview.innerHTML = `
            <div>Arbeitszeit: <b>${work.toFixed(2)} h</b></div>
            <div>Gesamt inkl. Bonus: <b>${total.toFixed(2)} h</b> × ${fmt(wage)} = <b>${fmt(cost)}</b></div>
            ${warn}
        `;
    }
}
```

- [ ] **Step 2: Neue Funktionen global exportieren** (`updateTimesheetModalDays`, `updateTimesheetLivePreview`)

- [ ] **Step 3: Browser-Verifikation**

1. Neues Timesheet (Admin), Modal öffnet.
2. Event wählen → Tag-Dropdown zeigt alle Tage.
3. Start/Ende/Pause ändern → Live-Preview rechnet mit.
4. Über-10h-Werte → rote Warnung.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(timesheet): rebuild openTimesheetModal with day select and live preview"
```

---

### Task 12: `saveTimesheet` mit breakMinutes, 10h-Confirm und Next-Day-Modus

**Files:**
- Modify: `index.html` — `saveTimesheet` (ca. Zeile 1488) ersetzen

- [ ] **Step 1: Funktion ersetzen**

```js
async function saveTimesheet(nextDay = false) {
    if (nextDay && nextDay.target) nextDay = true; // safety bei onclick
    const id = state.editTimesheetId || 'ts_' + Date.now();
    const eventId = $('ts-event-id').value, userId = $('ts-user-id').value;
    const date = $('ts-date').value, start = $('ts-start').value, end = $('ts-end').value;
    const breakMinutes = Number($('ts-break').value) || 0;
    const bonus = Number($('ts-bonus').value) || 0;
    if (\!eventId || \!userId || \!date || \!start || \!end)
        return showToast('Bitte alle Pflichtfelder ausfüllen\!', 'error');
    const user = find(state.users, userId, 'uid');
    const wage = user ? (Number(user.wage) || 14) : 14;
    const ev = find(state.events, eventId);
    const evStart = ev ? ev.date : date;
    const evEnd = ev ? (ev.endDate || ev.date) : date;
    const workHours = calculateHours(start, end, evStart, evEnd, breakMinutes);
    if (workHours <= 0)
        return showToast('Arbeitszeit ist 0 oder negativ (Pause zu lang?).', 'error');
    const totalHours = workHours + bonus;
    const daily = getDailyHours(userId, date, id) + totalHours;
    if (daily > 10) {
        const ok = confirm(
            `⚠ Warnung: ${user?.name || 'Mitarbeiter'} käme an ${fmtDate(date)} auf ${daily.toFixed(2)} h.\n` +
            `Das überschreitet die gesetzliche Höchstarbeitszeit von 10 h pro Tag.\n\n` +
            `Trotzdem speichern?`
        );
        if (\!ok) return;
    }
    const ts = {
        id, eventId, userId, date, start, end,
        breakMinutes: Number(breakMinutes),
        bonus: Number(bonus),
        workHours: Number(workHours.toFixed(2)),
        totalHours: Number(totalHours.toFixed(2)),
        wage: Number(wage),
        totalCost: Number((totalHours * wage).toFixed(2))
    };
    try {
        await set(ref(db, 'timesheets/' + id), ts);
        showToast('Zeiterfassung gespeichert\!', 'success');
        if (nextDay === true) {
            state.editTimesheetId = null;
            const days = ev ? eventDateRange(ev) : [];
            const idx = days.indexOf(date);
            if (idx >= 0 && idx < days.length - 1) {
                $('ts-date').value = days[idx + 1];
            }
            $('ts-start-wrap').innerHTML = timeSelectHTML('ts-start', '');
            $('ts-end-wrap').innerHTML   = timeSelectHTML('ts-end', '');
            $('ts-break-wrap').innerHTML = breakSelectHTML('ts-break', 0);
            $('ts-bonus-wrap').innerHTML = bonusSelectHTML('ts-bonus', 0);
            ['ts-start','ts-end','ts-break','ts-bonus'].forEach(id => {
                const el = $(id);
                if (el) el.onchange = updateTimesheetLivePreview;
            });
            updateTimesheetLivePreview();
        } else {
            closeModal('timesheet-modal');
        }
    } catch (e) {
        showToast('Fehler beim Speichern: ' + e.message, 'error');
    }
}
```

- [ ] **Step 2: Browser-Verifikation**

1. Neue Schicht anlegen, Event+User+Tag+Zeiten+Pause, Speichern.
2. Firebase: Eintrag mit `breakMinutes`, korrektem `workHours`.
3. Zweite Schicht am gleichen Tag so anlegen, dass Summe >10h: Confirm-Dialog kommt.
4. Abbrechen, kein Eintrag.
5. Mit "Speichern & nächster Tag": Tag springt auf nächsten.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(timesheet): saveTimesheet with breakMinutes, 10h confirm, next-day mode"
```

---

### Task 13: Mehrtages-Event-Karte (Selbst-Erfassung)

**Files:**
- Modify: `index.html` — `renderEventList` (ca. Zeile 1047–1063), `logHours` (ca. 1392)

- [ ] **Step 1: Neue Funktion `renderEventTimesheetBlock` einfügen**

```js
function renderEventTimesheetBlock(ev, uid) {
    const days = eventDateRange(ev);
    if (\!days.length) return '';
    const existingByDate = {};
    state.timesheets
        .filter(t => eq(t.eventId, ev.id) && eq(t.userId, uid))
        .forEach(t => { existingByDate[t.date] = t; });

    const rows = days.map(d => {
        const ts = existingByDate[d] || {};
        const startSel = timeSelectHTML(`ts-d-${esc(ev.id)}-${d}-start`, ts.start || '');
        const endSel   = timeSelectHTML(`ts-d-${esc(ev.id)}-${d}-end`,   ts.end   || '');
        const brkSel   = breakSelectHTML(`ts-d-${esc(ev.id)}-${d}-break`, ts.breakMinutes || 0);
        const bonusSel = bonusSelectHTML(`ts-d-${esc(ev.id)}-${d}-bonus`, ts.bonus || 0);
        const existingInfo = ts.id
            ? `<span class="muted">${Number(ts.totalHours).toFixed(2)} h · ${fmt(ts.totalCost)}</span>
               <button class="btn-sm btn-danger" onclick="deleteTimesheet('${escJs(ts.id)}')">🗑</button>`
            : '';
        return `
          <div class="ts-day-row" style="display:grid; grid-template-columns: auto 1fr 1fr 1fr 1fr auto auto; gap:6px; align-items:center; padding:4px 0; border-bottom:1px solid #f1f5f9;">
            <b>${esc(fmtDate(d))}</b>
            ${startSel} ${endSel} ${brkSel} ${bonusSel}
            <button class="btn-sm btn-primary"
                onclick="saveInlineTimesheet('${escJs(ev.id)}','${d}','${escJs(ts.id || '')}')">Speichern</button>
            ${existingInfo}
          </div>`;
    }).join('');

    const open = days.length <= 1 || new Date(ev.date) <= new Date(Date.now() + 7*86400000);
    return `
      <details ${open ? 'open' : ''} style="margin-top:8px;">
        <summary>Zeiterfassung (${days.length} Tag${days.length>1?'e':''})</summary>
        <div style="padding:6px 0;">${rows}</div>
      </details>`;
}
```

- [ ] **Step 2: Neue Funktion `saveInlineTimesheet` einfügen**

```js
async function saveInlineTimesheet(eventId, date, existingTsId) {
    const uid = auth.currentUser.uid;
    const start = $(`ts-d-${eventId}-${date}-start`).value;
    const end   = $(`ts-d-${eventId}-${date}-end`).value;
    const breakMinutes = Number($(`ts-d-${eventId}-${date}-break`).value) || 0;
    const bonus = Number($(`ts-d-${eventId}-${date}-bonus`).value) || 0;
    if (\!start || \!end) return showToast('Bitte Start und Ende auswählen\!', 'error');
    const ev = find(state.events, eventId);
    if (\!ev) return;
    const evEnd = ev.endDate || ev.date;
    const user = find(state.users, uid, 'uid');
    const wage = user ? (Number(user.wage) || 14) : 14;
    const workHours = calculateHours(start, end, ev.date, evEnd, breakMinutes);
    if (workHours <= 0) return showToast('Arbeitszeit ist 0 oder negativ.', 'error');
    const totalHours = workHours + bonus;
    const daily = getDailyHours(uid, date, existingTsId || null) + totalHours;
    if (daily > 10) {
        if (\!confirm(`⚠ Du käme am ${fmtDate(date)} auf ${daily.toFixed(2)} h (>10 h). Trotzdem speichern?`)) return;
    }
    const id = existingTsId || ('ts_' + Date.now() + '_' + uid);
    const ts = {
        id, eventId, userId: uid, date, start, end,
        breakMinutes, bonus,
        workHours: Number(workHours.toFixed(2)),
        totalHours: Number(totalHours.toFixed(2)),
        wage: Number(wage),
        totalCost: Number((totalHours * wage).toFixed(2))
    };
    try {
        await set(ref(db, 'timesheets/' + id), ts);
        showToast('Zeit gespeichert', 'success');
    } catch (e) {
        showToast('Fehler: ' + e.message, 'error');
    }
}
```

- [ ] **Step 3: `renderEventList` Inline-Erfassung ersetzen**

Die bisherigen `ts-start-`/`ts-end-`-Inputs in `renderEventList` (Zeile 1047–1063) durch `renderEventTimesheetBlock(ev, auth.currentUser.uid)` ersetzen. Den bisherigen `logHours`-Button entfernen.

- [ ] **Step 4: `logHours` entfernen oder als Alias beibehalten**

Per Grep prüfen, ob `logHours` an anderer Stelle referenziert wird. Falls nein: Funktion ersatzlos löschen. Falls doch: die Funktion leert durchreichen an `saveInlineTimesheet`.

- [ ] **Step 5: Alle neuen Funktionen global exportieren** (`saveInlineTimesheet`, `renderEventTimesheetBlock`)

- [ ] **Step 6: Browser-Verifikation**

1. Staff-Account, bestehendes Event: Event-Karte zeigt Zeiterfassung-Section.
2. Aufklappen: N Zeilen mit Dropdowns.
3. Werte, Speichern → Eintrag in Firebase.
4. Existierende Einträge zeigen rechts Summe + Papierkorb.
5. Mehrtägiges Event: alle Tage werden angezeigt.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(timesheet): multi-day inline timesheet entry per event day"
```

---

## 🚀 Push-Marker 2: Block 3 fertig

```bash
git push origin main
```

---

## Block 4 — Auswertungen und Detailansichten

### Task 14: `renderTimesheets` — Pausenspalte, 10h-Badge, klickbare Aggregat-Zeilen

**Files:**
- Modify: `index.html` — `renderTimesheets` (ca. Zeile 1420–1463) ersetzen

- [ ] **Step 1: Funktion ersetzen**

```js
function renderTimesheets() {
    if (\!isAdminRole()) return;
    const valid = state.timesheets.filter(ts => ts && ts.userId && ts.eventId);
    const eventStats = {}, userStats = {};
    const tsRows = [];
    valid.sort((a, b) => (b.date || "").localeCompare(a.date || "")).forEach(ts => {
        const user = find(state.users, ts.userId, 'uid') || { name: 'Gelöschter User', wage: 0 };
        const ev = find(state.events, ts.eventId) || { title: 'Gelöschtes Event' };
        const tHours = Number(ts.totalHours) || 0;
        const tCost = Number(ts.totalCost) || 0;
        const over = getDailyHours(ts.userId, ts.date) > 10;
        if (\!eventStats[ts.eventId]) eventStats[ts.eventId] = { id: ts.eventId, title: ev.title, totalHours: 0, totalCost: 0 };
        eventStats[ts.eventId].totalHours += tHours;
        eventStats[ts.eventId].totalCost += tCost;
        if (\!userStats[ts.userId]) userStats[ts.userId] = { id: ts.userId, name: user.name, totalHours: 0, totalCost: 0, events: new Set() };
        userStats[ts.userId].totalHours += tHours;
        userStats[ts.userId].totalCost += tCost;
        userStats[ts.userId].events.add(ev.title);
        tsRows.push(`<tr class="${over ? 'over-10h' : ''}">
            <td><b>${esc(ev.title)}</b></td>
            <td>${esc(user.name)}${over ? '<span class="over-10h-badge">⚠</span>' : ''}</td>
            <td>${esc(ts.date || '-')}</td>
            <td>${esc(ts.start || '-')} - ${esc(ts.end || '-')}</td>
            <td>${Number(ts.breakMinutes || 0)} min</td>
            <td>${tHours.toFixed(2)}h ${Number(ts.bonus) > 0 ? `(+${esc(ts.bonus)}h)` : ''}</td>
            <td>${fmt(tCost)}</td>
            <td>
                <button class="btn-sm" onclick="openTimesheetModal('${escJs(ts.id)}')">✏️</button>
                <button class="btn-sm btn-danger" onclick="deleteTimesheet('${escJs(ts.id)}')">🗑️</button>
                <button class="btn-sm btn-primary" onclick="exportSingleShift('${escJs(ts.id)}')">PDF</button>
            </td>
        </tr>`);
    });
    const lc = $('timesheet-list-container');
    if (lc) lc.innerHTML = buildTable(
        ['Event', 'Mitarbeiter', 'Datum', 'Zeit', 'Pause', 'Std', 'Lohn', 'Aktion'],
        tsRows, 'Keine erfassten Zeiten.');
    const evRows = Object.values(eventStats).map(es =>
        `<tr class="clickable-row" onclick="showEventDetail('${escJs(es.id)}')">
           <td><b>${esc(es.title)}</b></td>
           <td>${es.totalHours.toFixed(2)}h</td>
           <td><b style="color:var(--red)">${fmt(es.totalCost)}</b></td>
         </tr>`);
    const ec = $('timesheet-events-container');
    if (ec) ec.innerHTML = buildTable(['Event', 'Gesamtstunden', 'Gesamtkosten'], evRows, 'Keine Event-Daten.');
    const usrRows = Object.values(userStats).map(us =>
        `<tr class="clickable-row" onclick="showUserDetail('${escJs(us.id)}')">
           <td><b>${esc(us.name)}</b></td>
           <td>${us.totalHours.toFixed(2)}h</td>
           <td><b style="color:var(--green)">${fmt(us.totalCost)}</b></td>
           <td>${esc(Array.from(us.events).join(', '))}</td>
         </tr>`);
    const uc = $('timesheet-users-container');
    if (uc) uc.innerHTML = buildTable(['Mitarbeiter', 'Gesamtstunden', 'Gesamtlohn', 'Eingesetzt bei'], usrRows, 'Keine Mitarbeiter-Daten.');
}
```

- [ ] **Step 2: Platzhalter für noch nicht implementierte Funktionen setzen (Fallback, falls Task 16/18 noch nicht durch)**

```js
window.exportSingleShift = window.exportSingleShift || function(id){ showToast('PDF-Export folgt', 'info'); };
window.showUserDetail    = window.showUserDetail    || function(id){ showToast('Detail-Ansicht folgt', 'info'); };
```

- [ ] **Step 3: Browser-Verifikation**

1. Admin → Personal & Zeiten.
2. Pausen-Spalte zeigt 0 min bei alten Einträgen.
3. Über-10h-Fall: rote Zeile + ⚠-Badge.
4. Klick auf Event-Zeile: bestehendes `showEventDetail` öffnet.
5. Klick auf User-Zeile: Toast (Platzhalter).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(timesheet): render break column, 10h warning badge, clickable aggregate rows"
```

---

### Task 15: `showEventDetail` Kostenaufschlüsselung

**Files:**
- Modify: `index.html` — `showEventDetail` (ca. Zeile 1293–1340) erweitern

- [ ] **Step 1: `showEventDetail` kurz lesen und den Container finden, in den die Event-Details gerendert werden**

- [ ] **Step 2: Am Ende der Render-Kette (vor `openModal('event-detail-modal')`) den neuen Abschnitt anhängen**

```js
const tsList = timesheetsByEvent(eventId);
let tsHtml = '';
if (tsList.length) {
    let totalH = 0, totalC = 0;
    const daySet = new Set();
    const uidSet = new Set();
    const rows = tsList.map(t => {
        const u = find(state.users, t.userId, 'uid') || { name: 'Gelöscht' };
        const over = getDailyHours(t.userId, t.date) > 10;
        totalH += Number(t.totalHours) || 0;
        totalC += Number(t.totalCost) || 0;
        daySet.add(t.date);
        uidSet.add(t.userId);
        return `<tr class="${over ? 'over-10h' : ''}">
            <td>${esc(fmtDate(t.date))}</td>
            <td>${esc(u.name)}${over ? '<span class="over-10h-badge">⚠</span>' : ''}</td>
            <td>${esc(t.start)}–${esc(t.end)}</td>
            <td>${Number(t.breakMinutes||0)} min</td>
            <td>${Number(t.workHours||0).toFixed(2)} h</td>
            <td>${Number(t.bonus||0).toLocaleString('de-DE')}</td>
            <td>${Number(t.totalHours||0).toFixed(2)} h</td>
            <td>${fmt(t.wage)}</td>
            <td><b>${fmt(t.totalCost)}</b></td>
            <td>
              <button class="btn-sm" onclick="openTimesheetModal('${escJs(t.id)}')">✏️</button>
              <button class="btn-sm btn-danger" onclick="deleteTimesheet('${escJs(t.id)}')">🗑️</button>
            </td>
        </tr>`;
    }).join('');
    tsHtml = `
      <h3 style="margin-top:20px;">Zeiterfassung & Kosten</h3>
      <table>
        <thead><tr>
          <th>Datum</th><th>Mitarbeiter</th><th>Start–Ende</th><th>Pause</th>
          <th>Arbeit</th><th>Bonus</th><th>Gesamt</th><th>Lohn</th><th>Kosten</th><th></th>
        </tr></thead>
        <tbody>${rows}</tbody>
      </table>
      <div style="margin-top:10px; padding:10px; background:#f8fafc; border-radius:6px;">
        <b>Gesamtstunden:</b> ${totalH.toFixed(2)} h &nbsp;|&nbsp;
        <b>Gesamtkosten:</b> ${fmt(totalC)} &nbsp;|&nbsp;
        <b>Einsatztage:</b> ${daySet.size} &nbsp;|&nbsp;
        <b>Crew:</b> ${uidSet.size} Personen &nbsp;|&nbsp;
        <b>Ø/Tag:</b> ${fmt(totalC / Math.max(1, daySet.size))}
      </div>
      <div style="margin-top:10px;">
        <button class="btn btn-primary" onclick="exportEvent('${escJs(eventId)}')">📄 PDF Event-Export</button>
        <button class="btn" onclick="openTimesheetModalForEvent('${escJs(eventId)}')">+ Zeiterfassung hinzufügen</button>
      </div>
    `;
} else {
    tsHtml = `<p class="muted" style="margin-top:20px;">Noch keine Zeiten erfasst.</p>
      <button class="btn btn-primary" onclick="openTimesheetModalForEvent('${escJs(eventId)}')">+ Zeiterfassung hinzufügen</button>`;
}
// An den Modal-Content-Container anhängen: tsHtml per innerHTML += ... in die richtige Stelle einfügen.
```

- [ ] **Step 3: Helper `openTimesheetModalForEvent` einfügen**

```js
function openTimesheetModalForEvent(eventId) {
    openTimesheetModal(null);
    $('ts-event-id').value = eventId;
    updateTimesheetModalDays();
    updateTimesheetLivePreview();
}
```

- [ ] **Step 4: Platzhalter `exportEvent` setzen (wird Task 18)**

```js
window.exportEvent = window.exportEvent || function(id){ showToast('PDF-Export folgt', 'info'); };
```

- [ ] **Step 5: `openTimesheetModalForEvent` global exportieren**

- [ ] **Step 6: Browser-Verifikation**

1. Event-Liste → Event klicken → Modal mit neuer Kostentabelle.
2. "+ Zeiterfassung hinzufügen" öffnet Timesheet-Modal mit Event vorbelegt.
3. Zeilen-Edit-Icon öffnet Edit-Modal.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(events): cost breakdown table in event detail modal"
```

---

### Task 16: Neues Modal `user-detail-modal` + `showUserDetail`

**Files:**
- Modify: `index.html` — neues Modal-Markup + neue Funktionen

- [ ] **Step 1: Modal-Markup einfügen**

```html
<div id="user-detail-modal" class="modal">
  <div class="modal-content">
    <span class="close" onclick="closeModal('user-detail-modal')">&times;</span>
    <h2 id="ud-title">Mitarbeiter-Detail</h2>
    <div id="ud-meta" class="muted" style="margin-bottom:14px;"></div>
    <div class="form-row" style="display:flex; gap:8px; align-items:center; flex-wrap:wrap;">
      <label>Zeitraum:</label>
      <input type="month" id="ud-month">
      <span>oder</span>
      <input type="date" id="ud-from">
      bis
      <input type="date" id="ud-to">
      <button class="btn btn-sm" onclick="refreshUserDetail()">Übernehmen</button>
    </div>
    <div id="ud-body"></div>
    <div id="ud-summary" style="margin-top:12px; padding:10px; background:#f8fafc; border-radius:6px;"></div>
    <div style="margin-top:10px;">
      <button class="btn btn-primary" onclick="exportUserMonthFromDetail()">📄 PDF Monat</button>
      <button class="btn btn-primary" onclick="exportUserRangeFromDetail()">📄 PDF Zeitraum</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Funktionen einfügen**

```js
let _udCurrentUid = null;

function showUserDetail(uid) {
    _udCurrentUid = uid;
    const u = find(state.users, uid, 'uid');
    if (\!u) return showToast('Mitarbeiter nicht gefunden', 'error');
    $('ud-title').textContent = u.name || u.email || 'Mitarbeiter';
    $('ud-meta').innerHTML = `${esc(u.email || '')} · ${esc(u.phone || '')} · ${fmt(u.wage || 0)}/h`;
    const d = new Date();
    $('ud-month').value = `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}`;
    $('ud-from').value = '';
    $('ud-to').value = '';
    refreshUserDetail();
    openModal('user-detail-modal');
}

function getUserDetailRange() {
    const from = $('ud-from').value;
    const to   = $('ud-to').value;
    if (from && to) return { from, to };
    const m = $('ud-month').value;
    if (m) {
        const [y, mo] = m.split('-').map(Number);
        const last = new Date(y, mo, 0).getDate();
        return {
            from: `${y}-${String(mo).padStart(2,'0')}-01`,
            to:   `${y}-${String(mo).padStart(2,'0')}-${String(last).padStart(2,'0')}`
        };
    }
    return { from: null, to: null };
}

function refreshUserDetail() {
    const uid = _udCurrentUid;
    if (\!uid) return;
    const { from, to } = getUserDetailRange();
    const list = timesheetsByUser(uid, from, to);
    const groups = {};
    list.forEach(t => {
        if (\!groups[t.eventId]) groups[t.eventId] = [];
        groups[t.eventId].push(t);
    });
    let html = '';
    let totalH = 0, totalC = 0;
    const violationDays = new Set();
    Object.entries(groups).forEach(([eid, arr]) => {
        const ev = find(state.events, eid) || { title: 'Gelöschtes Event' };
        let evH = 0, evC = 0;
        const rows = arr.map(t => {
            const over = getDailyHours(t.userId, t.date) > 10;
            if (over) violationDays.add(t.date);
            evH += Number(t.totalHours) || 0;
            evC += Number(t.totalCost)  || 0;
            totalH += Number(t.totalHours) || 0;
            totalC += Number(t.totalCost) || 0;
            return `<tr class="clickable-row ${over ? 'over-10h' : ''}" onclick="openTimesheetModal('${escJs(t.id)}')">
                <td>${esc(fmtDate(t.date))}</td>
                <td>${esc(t.start)}–${esc(t.end)}</td>
                <td>${Number(t.breakMinutes||0)} min</td>
                <td>${Number(t.totalHours||0).toFixed(2)} h${over ? '<span class="over-10h-badge">⚠</span>' : ''}</td>
                <td>${fmt(t.totalCost)}</td>
            </tr>`;
        }).join('');
        html += `
          <details open style="margin-top:10px;">
            <summary><b>${esc(ev.title)}</b> · ${esc(ev.date || '')}${ev.endDate && ev.endDate \!== ev.date ? '–' + esc(ev.endDate) : ''}</summary>
            <table>
              <thead><tr><th>Datum</th><th>Zeit</th><th>Pause</th><th>Gesamt</th><th>Kosten</th></tr></thead>
              <tbody>${rows}</tbody>
              <tfoot><tr><td colspan="3"><b>Event-Σ</b></td><td><b>${evH.toFixed(2)} h</b></td><td><b>${fmt(evC)}</b></td></tr></tfoot>
            </table>
          </details>`;
    });
    if (\!list.length) html = `<p class="muted">Keine Einträge im Zeitraum.</p>`;
    $('ud-body').innerHTML = html;
    $('ud-summary').innerHTML = `
        <b>Gesamt:</b> ${totalH.toFixed(2)} h &nbsp;|&nbsp;
        <b>${fmt(totalC)}</b>
        ${violationDays.size ? ` &nbsp;|&nbsp; <span style="color:#b91c1c;"><b>10h-Verstöße:</b> ${violationDays.size} Tag${violationDays.size>1?'e':''}</span>` : ''}
    `;
}

function exportUserMonthFromDetail() {
    const uid = _udCurrentUid;
    const m = $('ud-month').value;
    if (\!uid || \!m) return showToast('Kein Monat ausgewählt', 'error');
    const [y, mo] = m.split('-').map(Number);
    exportUserMonth(uid, y, mo);
}

function exportUserRangeFromDetail() {
    const uid = _udCurrentUid;
    const { from, to } = getUserDetailRange();
    if (\!uid || \!from || \!to) return showToast('Kein Zeitraum', 'error');
    exportRange({ from, to, uid });
}
```

- [ ] **Step 3: Platzhalter für noch fehlende PDF-Exports**

```js
window.exportUserMonth = window.exportUserMonth || function(){ showToast('PDF-Export folgt','info'); };
window.exportRange     = window.exportRange     || function(){ showToast('PDF-Export folgt','info'); };
```

- [ ] **Step 4: Alle neuen Funktionen global exportieren** (`showUserDetail`, `refreshUserDetail`, `exportUserMonthFromDetail`, `exportUserRangeFromDetail`)

- [ ] **Step 5: Browser-Verifikation**

1. Personal & Zeiten → User-Aggregat-Zeile klicken → Detail-Modal öffnet.
2. Monat-Select ändert → Liste filtert.
3. Gruppierung pro Event, Zwischensummen.
4. 10h-Tage rot.
5. Klick auf Tag-Zeile öffnet Edit-Modal.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(users): user detail modal with per-event cost breakdown and range filter"
```

---

## 🚀 Push-Marker 3: Block 4 fertig

```bash
git push origin main
```

---

## Block 5 — PDF-Export (jsPDF)

### Task 17: Shared PDF-Helpers

**Files:**
- Modify: `index.html` — neuer Section-Block nach den Timesheet-Helpers

- [ ] **Step 1: Helpers einfügen**

```js
// ============================================================================
// PDF HELPERS (jsPDF)
// ============================================================================
function pdfDoc() {
    const { jsPDF } = window.jspdf || {};
    if (\!jsPDF) { showToast('jsPDF nicht geladen', 'error'); return null; }
    return new jsPDF({ unit: 'mm', format: 'a4', orientation: 'portrait' });
}

function pdfHeader(doc, title, subtitle = '') {
    if (LOGO_DATAURL) {
        try { doc.addImage(LOGO_DATAURL, 'PNG', 15, 10, 20, 20); } catch(e){}
    }
    doc.setFont('helvetica', 'bold');
    doc.setFontSize(16);
    doc.text('Heitmanns Pizza', 40, 18);
    doc.setFont('helvetica', 'normal');
    doc.setFontSize(9);
    doc.setTextColor(120);
    doc.text('Event-Catering · Hamburg', 40, 24);
    doc.setTextColor(0);
    doc.setFontSize(13);
    doc.setFont('helvetica', 'bold');
    doc.text(title, 15, 40);
    if (subtitle) {
        doc.setFont('helvetica', 'normal');
        doc.setFontSize(10);
        doc.setTextColor(80);
        doc.text(subtitle, 15, 46);
        doc.setTextColor(0);
    }
    doc.setDrawColor(200);
    doc.line(15, 50, 195, 50);
    return 56; // Y-Startposition für Content
}

function pdfFooter(doc) {
    const pageCount = doc.internal.getNumberOfPages();
    for (let i = 1; i <= pageCount; i++) {
        doc.setPage(i);
        doc.setFontSize(8);
        doc.setTextColor(120);
        doc.text(`Seite ${i} / ${pageCount}`, 15, 287);
        const now = new Date().toLocaleDateString('de-DE');
        doc.text(`Erstellt: ${now}`, 195, 287, { align: 'right' });
        doc.setTextColor(0);
    }
}

function pdfCurrency(n) {
    return (Number(n) || 0).toLocaleString('de-DE', { style: 'currency', currency: 'EUR' });
}
function pdfHours(n) {
    return (Number(n) || 0).toLocaleString('de-DE',
        { minimumFractionDigits: 2, maximumFractionDigits: 2 }) + ' h';
}
function pdfFilename(prefix, parts) {
    const slug = s => (s || '').toString().toLowerCase()
        .replace(/[äöüß]/g, m => ({'ä':'ae','ö':'oe','ü':'ue','ß':'ss'}[m]))
        .replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '');
    return `heitmanns-${prefix}-${parts.map(slug).filter(Boolean).join('-')}.pdf`;
}

const PDF_HEAD_STYLES = { fillColor: [200, 30, 30], textColor: 255, fontStyle: 'bold' };
const PDF_STYLES = { fontSize: 9, cellPadding: 2 };
```

- [ ] **Step 2: Browser-Verifikation (DevTools Console)**

```js
const d = pdfDoc();
pdfHeader(d, 'Test-Titel', 'Test-Subtitle');
pdfFooter(d);
d.save('test.pdf');
```

PDF wird heruntergeladen, Logo + Titel sichtbar, Seitenzahl im Footer.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(pdf): add shared jsPDF helpers (header, footer, formatters)"
```

---

### Task 18: Vier Export-Funktionen

**Files:**
- Modify: `index.html` — Export-Funktionen unter den PDF-Helpers

- [ ] **Step 1: `exportUserMonth` einfügen**

```js
function exportUserMonth(uid, year, month) {
    const user = find(state.users, uid, 'uid');
    if (\!user) return showToast('Mitarbeiter nicht gefunden', 'error');
    const mm = String(month).padStart(2, '0');
    const lastDay = new Date(year, month, 0).getDate();
    const from = `${year}-${mm}-01`;
    const to   = `${year}-${mm}-${String(lastDay).padStart(2,'0')}`;
    const list = timesheetsByUser(uid, from, to);
    const monthName = new Date(year, month - 1, 1)
        .toLocaleDateString('de-DE', { month: 'long', year: 'numeric' });
    const doc = pdfDoc();
    if (\!doc) return;
    const startY = pdfHeader(doc, `Zeiterfassung ${user.name || user.email}`, monthName);
    doc.setFontSize(9);
    doc.text(`Email: ${user.email || ''}`, 15, startY);
    doc.text(`Telefon: ${user.phone || '-'}`, 15, startY + 5);
    doc.text(`Stundenlohn: ${pdfCurrency(user.wage || 14)}`, 15, startY + 10);

    let totalH = 0, totalC = 0;
    const body = list.map(t => {
        totalH += Number(t.totalHours) || 0;
        totalC += Number(t.totalCost) || 0;
        const ev = find(state.events, t.eventId) || { title: '?' };
        const over = getDailyHours(t.userId, t.date) > 10;
        const row = [
            fmtDate(t.date), ev.title || '',
            t.start || '', t.end || '',
            `${Number(t.breakMinutes || 0)}`,
            pdfHours(t.workHours),
            Number(t.bonus || 0).toLocaleString('de-DE'),
            pdfHours(t.totalHours),
            pdfCurrency(t.wage),
            pdfCurrency(t.totalCost)
        ];
        row._over = over;
        return row;
    });
    doc.autoTable({
        startY: startY + 16,
        head: [['Datum','Event','Start','Ende','Pause','Arbeit','Bonus','Gesamt','Lohn','Kosten']],
        body,
        theme: 'grid',
        headStyles: PDF_HEAD_STYLES,
        styles: PDF_STYLES,
        columnStyles: {
            0: { halign: 'center' },
            5: { halign: 'right' },
            6: { halign: 'right' },
            7: { halign: 'right' },
            8: { halign: 'right' },
            9: { halign: 'right' }
        },
        didParseCell(data) {
            if (data.section === 'body' && data.row.raw && data.row.raw._over) {
                data.cell.styles.fillColor = [255, 230, 230];
            }
        },
        foot: [[
            { content: 'Summe', colSpan: 7, styles: { halign: 'right', fontStyle: 'bold' } },
            { content: pdfHours(totalH), styles: { halign: 'right', fontStyle: 'bold' } },
            '',
            { content: pdfCurrency(totalC), styles: { halign: 'right', fontStyle: 'bold' } }
        ]]
    });
    const y = doc.lastAutoTable.finalY + 20;
    doc.setFontSize(9);
    doc.text('_______________________', 15, y + 10);
    doc.text('Mitarbeiter', 15, y + 15);
    doc.text('_______________________', 80, y + 10);
    doc.text('Arbeitgeber', 80, y + 15);
    doc.text('_______________________', 145, y + 10);
    doc.text('Datum', 145, y + 15);
    pdfFooter(doc);
    doc.save(pdfFilename('ma', [user.name || uid, `${year}-${mm}`]));
}
```

- [ ] **Step 2: `exportEvent` einfügen**

```js
function exportEvent(eventId) {
    const ev = find(state.events, eventId);
    if (\!ev) return showToast('Event nicht gefunden', 'error');
    const list = timesheetsByEvent(eventId);
    const doc = pdfDoc();
    if (\!doc) return;
    const subtitle = ev.endDate && ev.endDate \!== ev.date
        ? `${fmtDate(ev.date)} – ${fmtDate(ev.endDate)}`
        : fmtDate(ev.date || '');
    const startY = pdfHeader(doc, `Event: ${ev.title || ''}`, subtitle);
    doc.setFontSize(9);
    doc.text(`Typ: ${ev.type || '-'}`, 15, startY);
    doc.text(`Ort: ${ev.ort || '-'}`, 15, startY + 5);
    doc.text(`Personen: ${ev.personen || '-'}`, 15, startY + 10);

    const byUser = {};
    list.forEach(t => {
        if (\!byUser[t.userId]) byUser[t.userId] = [];
        byUser[t.userId].push(t);
    });
    let totalH = 0, totalC = 0;
    const body = [];
    Object.entries(byUser).forEach(([uid, arr]) => {
        const u = find(state.users, uid, 'uid') || { name: 'Gelöscht' };
        let subH = 0, subC = 0;
        arr.forEach(t => {
            subH += Number(t.totalHours) || 0;
            subC += Number(t.totalCost) || 0;
            totalH += Number(t.totalHours) || 0;
            totalC += Number(t.totalCost) || 0;
            const over = getDailyHours(t.userId, t.date) > 10;
            const row = [
                u.name || '',
                fmtDate(t.date),
                `${t.start}–${t.end}`,
                `${Number(t.breakMinutes || 0)} min`,
                pdfHours(t.workHours),
                pdfHours(t.totalHours),
                pdfCurrency(t.totalCost)
            ];
            row._over = over;
            body.push(row);
        });
        body.push([
            { content: `Σ ${u.name}`, colSpan: 5, styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
            { content: pdfHours(subH), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
            { content: pdfCurrency(subC), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } }
        ]);
    });
    doc.autoTable({
        startY: startY + 18,
        head: [['Mitarbeiter','Datum','Zeit','Pause','Arbeit','Gesamt','Kosten']],
        body,
        theme: 'grid',
        headStyles: PDF_HEAD_STYLES,
        styles: PDF_STYLES,
        didParseCell(data) {
            if (data.section === 'body' && data.row.raw && data.row.raw._over) {
                data.cell.styles.fillColor = [255, 230, 230];
            }
        },
        foot: [[
            { content: 'Event-Summe', colSpan: 5, styles: { halign: 'right', fontStyle: 'bold' } },
            { content: pdfHours(totalH), styles: { halign: 'right', fontStyle: 'bold' } },
            { content: pdfCurrency(totalC), styles: { halign: 'right', fontStyle: 'bold' } }
        ]]
    });
    pdfFooter(doc);
    doc.save(pdfFilename('event', [ev.title || eventId, ev.date || '']));
}
```

- [ ] **Step 3: `exportRange` einfügen**

```js
function exportRange({ from, to, uid = null, eventId = null }) {
    if (\!from || \!to) return showToast('Zeitraum fehlt', 'error');
    let list = state.timesheets.filter(t => t && t.date >= from && t.date <= to);
    if (uid)     list = list.filter(t => eq(t.userId, uid));
    if (eventId) list = list.filter(t => eq(t.eventId, eventId));
    list.sort((a, b) => (a.date || '').localeCompare(b.date || ''));
    const doc = pdfDoc();
    if (\!doc) return;
    const subtitleParts = [`${fmtDate(from)} – ${fmtDate(to)}`];
    if (uid) {
        const u = find(state.users, uid, 'uid');
        subtitleParts.push(`Mitarbeiter: ${u ? u.name : uid}`);
    }
    if (eventId) {
        const ev = find(state.events, eventId);
        subtitleParts.push(`Event: ${ev ? ev.title : eventId}`);
    }
    const startY = pdfHeader(doc, 'Zeiterfassung Zeitraum', subtitleParts.join(' · '));
    let totalH = 0, totalC = 0;
    const body = list.map(t => {
        totalH += Number(t.totalHours) || 0;
        totalC += Number(t.totalCost) || 0;
        const u = find(state.users, t.userId, 'uid') || { name: '?' };
        const ev = find(state.events, t.eventId) || { title: '?' };
        const over = getDailyHours(t.userId, t.date) > 10;
        const row = [
            fmtDate(t.date),
            u.name || '',
            ev.title || '',
            `${t.start}–${t.end}`,
            `${Number(t.breakMinutes || 0)}`,
            pdfHours(t.totalHours),
            pdfCurrency(t.totalCost)
        ];
        row._over = over;
        return row;
    });
    doc.autoTable({
        startY: startY + 6,
        head: [['Datum','Mitarbeiter','Event','Zeit','Pause','Gesamt','Kosten']],
        body,
        theme: 'grid',
        headStyles: PDF_HEAD_STYLES,
        styles: PDF_STYLES,
        didParseCell(data) {
            if (data.section === 'body' && data.row.raw && data.row.raw._over) {
                data.cell.styles.fillColor = [255, 230, 230];
            }
        },
        foot: [[
            { content: 'Summe', colSpan: 5, styles: { halign: 'right', fontStyle: 'bold' } },
            { content: pdfHours(totalH), styles: { halign: 'right', fontStyle: 'bold' } },
            { content: pdfCurrency(totalC), styles: { halign: 'right', fontStyle: 'bold' } }
        ]]
    });
    pdfFooter(doc);
    doc.save(pdfFilename('range', [from, to]));
}
```

- [ ] **Step 4: `exportSingleShift` einfügen und `printReceipt` umleiten**

```js
function exportSingleShift(tsId) {
    const ts = find(state.timesheets, tsId);
    if (\!ts) return showToast('Eintrag nicht gefunden', 'error');
    const ev = find(state.events, ts.eventId) || { title: '?', type: '-', date: ts.date };
    const u  = find(state.users, ts.userId, 'uid') || { name: '?' };
    const doc = pdfDoc();
    if (\!doc) return;
    const startY = pdfHeader(doc, `Einzel-Schicht`, `${u.name} · ${fmtDate(ts.date)}`);
    doc.setFontSize(11);
    const lines = [
        ['Mitarbeiter:',       u.name || ''],
        ['Event:',             `${ev.title} (${ev.type || '-'})`],
        ['Datum:',             fmtDate(ts.date)],
        ['Start–Ende:',        `${ts.start} – ${ts.end}`],
        ['Pause:',             `${Number(ts.breakMinutes || 0)} min`],
        ['Arbeitszeit:',       pdfHours(ts.workHours)],
        ['Bonus:',             `${Number(ts.bonus || 0).toLocaleString('de-DE')} h`],
        ['Gesamtstunden:',     pdfHours(ts.totalHours)],
        ['Stundenlohn:',       pdfCurrency(ts.wage)],
        ['Auszahlungsbetrag:', pdfCurrency(ts.totalCost)]
    ];
    let y = startY + 4;
    lines.forEach(([k, v]) => {
        doc.setFont('helvetica', 'bold');
        doc.text(k, 15, y);
        doc.setFont('helvetica', 'normal');
        doc.text(v, 70, y);
        y += 7;
    });
    pdfFooter(doc);
    doc.save(pdfFilename('schicht', [u.name || '', ts.date || '']));
}

// printReceipt als Alias behalten
function printReceipt(tsId) { return exportSingleShift(tsId); }
```

- [ ] **Step 5: Alle Export-Funktionen global exportieren** (`exportUserMonth`, `exportEvent`, `exportRange`, `exportSingleShift`, `printReceipt`)

- [ ] **Step 6: Browser-Verifikation**

1. Personal & Zeiten → User-Zeile → Detail → "PDF Monat" → PDF ok.
2. Spalten, Logo, Summen, Footer-Seitenzahl prüfen.
3. Event-Detail → "PDF Event-Export" → Gruppierung pro Mitarbeiter.
4. User-Detail "PDF Zeitraum" → PDF.
5. Timesheet-Liste → "PDF"-Button pro Zeile → Einzel-Schicht-PDF.
6. Über-10h-Zeilen sind in jedem PDF rosa hinterlegt.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(pdf): implement user month, event, range, single shift exports"
```

---

### Task 19: Zeitraum-Export-Modal und Tab-Button

**Files:**
- Modify: `index.html` — neues Modal + Button im Personal & Zeiten Tab

- [ ] **Step 1: Modal-Markup**

```html
<div id="range-export-modal" class="modal">
  <div class="modal-content admin-only">
    <span class="close" onclick="closeModal('range-export-modal')">&times;</span>
    <h2>Zeitraum-Export</h2>
    <div class="form-row"><label>Von</label><input type="date" id="re-from"></div>
    <div class="form-row"><label>Bis</label><input type="date" id="re-to"></div>
    <div class="form-row"><label>Mitarbeiter (optional)</label>
      <select id="re-user"><option value="">Alle</option></select></div>
    <div class="form-row"><label>Event (optional)</label>
      <select id="re-event"><option value="">Alle</option></select></div>
    <div class="btn-row">
      <button class="btn btn-primary" onclick="runRangeExport()">📄 Exportieren</button>
      <button class="btn" onclick="closeModal('range-export-modal')">Abbrechen</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Funktionen einfügen**

```js
function openRangeExportModal() {
    $('re-user').innerHTML = '<option value="">Alle</option>' +
        state.users.map(u => `<option value="${esc(u.uid)}">${esc(u.name || u.email)}</option>`).join('');
    $('re-event').innerHTML = '<option value="">Alle</option>' +
        state.events.map(e => `<option value="${esc(e.id)}">${esc(e.date)} · ${esc(e.title)}</option>`).join('');
    const now = new Date();
    const first = new Date(now.getFullYear(), now.getMonth(), 1);
    $('re-from').value = first.toISOString().slice(0, 10);
    $('re-to').value   = now.toISOString().slice(0, 10);
    openModal('range-export-modal');
}
function runRangeExport() {
    const from = $('re-from').value, to = $('re-to').value;
    if (\!from || \!to) return showToast('Zeitraum fehlt', 'error');
    exportRange({
        from, to,
        uid:     $('re-user').value || null,
        eventId: $('re-event').value || null
    });
    closeModal('range-export-modal');
}
```

- [ ] **Step 3: Button im Personal & Zeiten Tab einfügen**

```html
<button class="btn btn-primary" onclick="openRangeExportModal()">📄 Zeitraum-Export</button>
```

Platzierung: nahe dem bestehenden "Neue Zeiterfassung"-Button.

- [ ] **Step 4: Funktionen global exportieren**

- [ ] **Step 5: Browser-Verifikation**

1. Personal & Zeiten → Zeitraum-Export-Button → Modal.
2. Zeitraum + optionale Filter → Exportieren → PDF öffnet.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(pdf): add range export modal with user/event filters"
```

---

## 🚀 Push-Marker 4: Block 5 fertig

```bash
git push origin main
```

---

## Block 6 — Abschluss

### Task 20: Manuelle Test-Runde nach Spec-Checklist

**Files:** Keine Änderungen, reines Testing gegen `https://arndtbockhop.github.io/HeitmannsPizza-Dashboard/`. GitHub Pages deployed nach einem Push in ca. 1–2 Minuten.

- [ ] **Step 1: Datenmodell und Helpers**
  - `calculateHours('18:00','23:30','2026-04-14','2026-04-14', 30)` → 5,00
  - `calculateHours('22:00','02:00','2026-04-14','2026-04-15', 0)` → 4,00
  - `eventDateRange({date:'2026-04-14', endDate:'2026-04-16'})` → 3 Einträge
  - `getDailyHours(existingUid, '2026-04-14', 'nonexistent')` zählt nicht doppelt

- [ ] **Step 2: Mitarbeiter-Flow**
  - Neuer Mitarbeiter anlegen (alle Felder)
  - Admin bleibt eingeloggt
  - Bearbeiten funktioniert
  - Mailto öffnet korrekt
  - Self-Login matcht Email, alter Datensatz verschwindet

- [ ] **Step 3: Zeiterfassung**
  - Eintages-Event: eine Zeile
  - Dreitages-Event: drei Zeilen
  - Dropdowns 15-min
  - Speichern funktioniert, Edit-Modus lädt Werte
  - Pause > Arbeitszeit → Toast-Fehler
  - Admin-Modal "nächster Tag"-Button

- [ ] **Step 4: Detailansichten**
  - Event-Detail → Kostentabelle, Summen korrekt
  - User-Detail → Gruppierung pro Event, Filter funktioniert
  - Tag-Zeile klickbar → Edit-Modal

- [ ] **Step 5: 10h-Warnung**
  - Confirm beim Speichern
  - Abbrechen → kein Save
  - Rote Zeilen in Listen
  - Verstoß-Zähler in User-Detail

- [ ] **Step 6: PDF-Export**
  - Mitarbeiter-Monat
  - Event-Export
  - Zeitraum-Export (alle drei Filter-Kombinationen)
  - Einzel-Schicht
  - Alle mit Logo, Summen, Footer-Seitenzahl

- [ ] **Step 7: Regression**
  - Bestehender `printReceipt`-Button funktioniert weiter
  - Alte Timesheets ohne `breakMinutes` werden mit 0 min angezeigt
  - Alte User ohne neue Felder werden ohne Fehler gerendert

- [ ] **Step 8: Abschluss-Commit für Testdaten-Aufräumen**

Alle Test-User und Test-Timesheets aus Firebase manuell löschen.

```bash
git commit --allow-empty -m "test: manual verification passed for zeiterfassung rework"
git push origin main
```

---

## Self-Review-Ergebnis (vom Plan-Autor beim Schreiben erledigt)

**Spec-Coverage:**
- Mehrtages-Zeiterfassung + 15-min-Dropdowns → Tasks 10–13
- Mitarbeiter anlegen + Felder → Tasks 4–9
- Event-Detail mit Kosten → Task 15
- Mitarbeiter-Detail → Task 16
- 10h-Warnung → Tasks 11, 12, 14, 16
- PDF-Export (4 Formate) → Tasks 17–19
- Datenmodell-Änderungen (breakMinutes, user-Felder, helpers) → Tasks 1–3
- Migration (additiv, keine Migration nötig) → in Tasks 2, 9, 14 explizit berücksichtigt

**Placeholder-Scan:** Keine `TBD`/`TODO`. Alle Code-Blöcke komplett. Zeile-Nummern-Hinweise sind als "ca. Zeile X" markiert, weil die Datei beim Implementieren wächst. Section-Anchors (`renderSettings`, `saveTimesheet`, `openTimesheetModal`, `renderEventList`) sind die stabilen Anker.

**Type-Consistency:**
- `breakMinutes` als Zahl konsequent
- `canLogin` als Boolean konsequent
- `createdByAdmin` als Boolean konsequent
- PDF-Funktionsnamen: `exportUserMonth`, `exportEvent`, `exportRange`, `exportSingleShift` — gleich in Tasks 14, 15, 16, 18, 19
- `openTimesheetModalForEvent` in Task 15 definiert und aufgerufen ✓
- `timesheetsByEvent` / `timesheetsByUser` in Task 2 definiert, ab Task 15/16 genutzt ✓
- `getDailyHours(uid, date, excludeTsId?)` Signatur in Task 2 etabliert, konsistent aufgerufen in Tasks 11, 12, 13, 14, 15, 16, 18

**Risiko-Punkte:**
- `logHours`-Legacy in Task 13: wenn Grep zeigt, dass `logHours` nirgends mehr referenziert ist, die Funktion ersatzlos löschen.
- Task 6 (Button in `renderSettings`) bleibt etwas generisch, weil der konkrete Render-String in `renderSettings` beim Implementieren gelesen wird. Explizit angemerkt.
- Task 9 (`updateUser`): der bestehende Save-Block variiert. Vor der Implementierung kurz `updateUser` lesen und die `updates`-Struktur ergänzen.
- Task 15 (`showEventDetail`): der richtige Container fürs Anhängen der Kostenaufschlüsselung muss beim Lesen identifiziert werden.

**XSS-Hinweis:** Alle Template-Strings nutzen `esc()` und `escJs()` für User-/DB-Eingaben, entsprechend dem bereits etablierten Pattern in `index.html`. Das ist die projektweite Sanitization-Strategie.

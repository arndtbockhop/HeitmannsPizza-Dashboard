# Zeiterfassung & Auswertung — Rework

**Datum:** 2026-04-13
**Projekt:** HeitmannsPizza-Dashboard
**Stack:** Single-file `index.html`, Vanilla JS ES-Modules, Firebase Realtime DB (Region europe-west1), GitHub Pages
**Scope:** Zeiterfassung mehrtagestauglich, Mitarbeiter-Stammdatenverwaltung, Detail-Auswertungen, 10h-Warnung, PDF-Export mit jsPDF

---

## Hintergrund

Das bestehende Dashboard unterstützt eine minimale Zeiterfassung (ein Timesheet pro Event/Person/Tag), hat aber kein UI für mehrtägige Events, keine 15-min-Rasterung, kein Mitarbeiter-Anlegen-Formular, keine Detail-Auswertung beim Klick auf Events/Mitarbeiter und nur eine rudimentäre Einzel-Quittung als "PDF" via `window.print()`. Eine Excel-Vorlage (`Zeiterfassung - Mitarbeiter.xlsx`) dient als Logik-Referenz: pro Zeile ein Mitarbeiter × Tag × Event, mit Bonus als Stunden-Dezimal, drei Aggregat-Sheets (Event, Mitarbeiter, Matrix).

**Gute Nachricht:** Das Datenmodell passt fast perfekt. Timesheets sind bereits 1 Tag = 1 Eintrag, Events haben bereits `date` + `endDate`. Die Änderungen sind überwiegend **additiv**, Migration entfällt.

---

## Ziele

1. **Mehrtägige Zeiterfassung** — Ein Event mit `endDate > date` rendert pro Tag eine eigene Eingabezeile. Admin und Mitarbeiter tragen jeden Tag separat ein.
2. **15-Minuten-Rasterung** — Start, Ende und Pause werden als Dropdowns mit 15-min-Schritten geführt (96 Slots für Zeit, feste Liste für Pausen).
3. **Mitarbeiter anlegen** — Admin-Formular in Settings, legt User-Datensatz in Firebase DB an (ohne Firebase-Auth-Aktion, damit der Admin eingeloggt bleibt). Matching beim späteren Self-Login per Email.
4. **Event-Detailansicht mit Kostenaufschlüsselung** — Klick auf Event in der Auswertung zeigt Modal mit Tabelle aller Timesheets, Summen, Gesamtkosten.
5. **Mitarbeiter-Detailansicht** — Analoges Modal mit Filter auf Zeitraum, gruppiert nach Event, Zwischensummen.
6. **10h-Warnung** — Warnung beim Speichern (confirm), rote Markierung in allen Listen und Auswertungen, Zähler in User-Detail-Summe.
7. **PDF-Export mit jsPDF** — Vier Formate: pro Mitarbeiter/Monat, pro Event, freier Zeitraum, Einzel-Schicht (Abwärtskompatibilität).
8. **Pausenfeld** — Optionales Feld in Minuten, wird von der Arbeitszeit abgezogen. Default 0.

## Non-Ziele

- Kein Matrix-Kalender-Grid (Wochen/Monatsansicht).
- Keine Trinkgeld-Umstellung von Stunden auf Euro (User-Entscheidung, bleibt wie Excel).
- Keine Mitarbeiter-Self-Availability für Events (eigenes Folge-Feature, separat geplant).
- Kein Build-Schritt, keine Frontend-Framework-Migration. Alles bleibt in `index.html` + CDN-Scripts.

---

## Architektur

**Ansatz 1 aus dem Brainstorming:** minimal-invasiv, alle Erweiterungen innerhalb von `index.html`. Einzige neue Abhängigkeiten: `jsPDF` + `jspdf-autotable` via CDN (`jsdelivr`), in `<head>` eingebunden. ~30 kB gzipped, kein Build-Schritt.

Bestehende Patterns bleiben unangetastet:
- `state`-Objekt + `onValue`-Listener (Zeile 800–886)
- `dbAction(...)` mit Toast-Feedback
- `renderXxx` / `openModal` / `closeModal`
- Globale Funktions-Exporte per `Object.assign(window, {...})` am Ende

---

## Datenmodell (Änderungen)

### `users/{uid}` — neue Felder (alle optional, additiv)

```
{
  // bestehend:
  email, name, role, wage, cleaningPoints, notifications,

  // NEU:
  phone:          string,   // ""
  address:        string,   // ""
  birthDate:      string,   // "YYYY-MM-DD"
  hireDate:       string,   // "YYYY-MM-DD"
  notes:          string,   // ""
  canLogin:       boolean,  // default true
  createdByAdmin: boolean   // default false, flag für Login-Matching
}
```

Bestehende User: fehlende Felder werden beim Render mit `?? ''` behandelt.

### `timesheets/{id}` — ein neues Feld

```
{
  // bestehend:
  id, eventId, userId, date, start, end, bonus,
  workHours, totalHours, wage, totalCost,

  // NEU:
  breakMinutes: number   // default 0
}
```

**Neue Berechnung:**

```
workHours  = ((end − start) in h) − (breakMinutes / 60)
totalHours = workHours + bonus     // bonus bleibt Stunden-Dezimal (Excel-Modell)
totalCost  = totalHours × wage
```

### `events/{id}` — unverändert

`date` + `endDate` existieren bereits. Ein-Tages-Events: `endDate === date` (oder fehlt → Fallback auf `date`).

### Neue Helper-Funktionen

| Helper | Zweck |
|---|---|
| `eventDateRange(ev) → string[]` | Array aller Tage zwischen `date` und `endDate` inklusiv |
| `getDailyHours(uid, dateStr, excludeTsId?) → number` | Summe `totalHours` aller Timesheets eines Users an einem Tag, optional ohne einen bestimmten Eintrag (für Edit-Fall) |
| `isOver10h(uid, dateStr, excludeTsId?) → boolean` | `getDailyHours > 10` |
| `timesheetsByEvent(eventId) → Timesheet[]` | Filter + Sortierung nach Datum, dann Userame |
| `timesheetsByUser(uid, fromDate?, toDate?) → Timesheet[]` | Filter auf User und optionalen Range |
| `calculateHours(start, end, date1, date2, breakMinutes = 0)` | erweiterte Version, abwärtskompatibel |

---

## Feature 1 — Mitarbeiter anlegen

### UI-Platzierung
Tab "Mitarbeiter Handling" in Settings (`renderSettings`, aktuell Zeile 2090). Button **"+ Neuer Mitarbeiter"** über der Team-Liste öffnet neues Modal `#user-create-modal`.

### Formularfelder

| Feld | Typ | Pflicht | Default |
|---|---|---|---|
| Name | text | ✓ | — |
| Email | email | ✓ | — |
| Telefon | tel | — | — |
| Adresse | text | — | — |
| Geburtsdatum | date | — | — |
| Eintrittsdatum | date | — | — |
| Rolle | select (staff / admin / superadmin) | ✓ | staff |
| Default-Stundenlohn | number, step 0.5, €/h | ✓ | 14 |
| Notiz | textarea | — | — |
| ☑ Kann sich einloggen | checkbox | — | true |

### Save-Logik

```js
async function saveNewUser(form) {
  const newId = 'u_' + Date.now();
  const data = {
    name, email, phone, address, birthDate, hireDate,
    role, wage, notes, canLogin,
    createdByAdmin: true,
    cleaningPoints: 0
  };
  await dbAction(set(ref(db, 'users/' + newId), data),
                 'Mitarbeiter angelegt');
  closeModal('user-create-modal');
}
```

**Wichtig:** Kein `createUserWithEmailAndPassword` — würde den Admin ausloggen. Der Datensatz liegt unter einer vom Admin generierten DB-ID (`u_<timestamp>`), nicht unter einer Firebase-Auth-UID.

### Login-Matching

Der bestehende `doLogin`-Flow (Zeile 771–789) wird erweitert:

```js
// Nach erfolgreichem signInWithEmailAndPassword:
let userDbRef = ref(db, 'users/' + auth.currentUser.uid);
let snap = await get(userDbRef);
if (!snap.exists()) {
  // Suche nach admin-angelegtem User mit gleicher Email:
  const allUsers = await get(ref(db, 'users'));
  const match = Object.entries(allUsers.val() || {})
    .find(([id, u]) =>
      u.createdByAdmin === true &&
      u.email?.toLowerCase() === auth.currentUser.email?.toLowerCase()
    );
  if (match) {
    const [oldId, oldData] = match;
    // Atomar: neuen Datensatz anlegen, alten löschen
    await update(ref(db), {
      ['users/' + auth.currentUser.uid]: {
        ...oldData, createdByAdmin: false
      },
      ['users/' + oldId]: null
    });
  } else {
    // Fallback: auto-create wie bisher
  }
}
```

### "Zugangsdaten senden" — Mailto

In der Team-Liste, neben jedem User mit `createdByAdmin === true && canLogin === true`, ein 📧-Icon-Button:

```js
function sendCredentials(user) {
  const subject = encodeURIComponent('Zugang HeitmannsPizza-Dashboard');
  const body = encodeURIComponent(
`Hi ${user.name},

Du kannst Dich ab sofort im Dashboard einloggen:
https://arndtbockhop.github.io/HeitmannsPizza-Dashboard/

Email: ${user.email}
PIN: Beim ersten Login wählst Du eine 4-stellige PIN.

Viele Grüße
Heitmanns Pizza`);
  location.href = `mailto:${user.email}?subject=${subject}&body=${body}`;
}
```

Kein Resend, kein SMTP-Backend. Öffnet das Mail-Programm des Admins.

### Bearbeiten & Löschen
Bestehende `updateUser` (Zeile 2165) um die neuen Felder erweitern. Delete-Button mit `confirm()`. Self-Delete und Self-Role-Change bleiben gesperrt wie bisher.

---

## Feature 2 — Zeiterfassung Eingabe

### 15-min-Slots (einmalig berechnet)

```js
const TIME_SLOTS = (() => {
  const arr = [];
  for (let h = 0; h < 24; h++)
    for (let m = 0; m < 60; m += 15)
      arr.push(`${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')}`);
  return arr;  // 96 Einträge
})();

const BREAK_OPTIONS = [0, 15, 30, 45, 60, 75, 90, 120];
const BONUS_OPTIONS = [0, 0.25, 0.5, 0.75, 1];
```

Helper `timeSelect(id, selected)`, `breakSelect(id, selected)`, `bonusSelect(id, selected)` rendern reusable `<select>`-Elemente.

### Event-Karte — Mitarbeiter-Selbst-Erfassung

Die bisherige Inline-Erfassung (`renderEventList`, Zeile 1047–1063) wird ersetzt. Pro Tag aus `eventDateRange(ev)` wird eine Zeile gerendert, eingefasst in ein `<details>`-Element (default offen bei Events ≤ 7 Tage in Zukunft):

```
Di, 14.04.2026  Start [18:00▾]  Ende [23:30▾]
                Pause [30 ▾] min  Bonus [0,25 ▾] h
                → 5,25 h · 73,50 €     [Speichern]
```

Wenn für den Tag bereits ein Timesheet existiert, wird die Zeile im **Edit-Modus** gerendert (Werte vorbefüllt, Speichern = Update, zusätzlich Lösch-Icon).

### Admin-Modal (`#timesheet-modal`, Zeile 556)

```
Event:       [Dropdown ▾]
Mitarbeiter: [Dropdown ▾]
────────────────────────
Tag:         [Dropdown ▾]   ← Tage des gewählten Events
Start:       [18:00 ▾]
Ende:        [23:30 ▾]
Pause:       [30 ▾] min
Bonus:       [0,25 ▾] h
────────────────────────
Arbeitszeit: 5,00 h             (live berechnet)
Gesamt:      5,25 h × 14,00 € = 73,50 €
⚠ Dieser Tag überschreitet 10 h  (falls zutreffend)

[Speichern]  [Speichern & nächster Tag]
```

"Speichern & nächster Tag" leert nur Tag/Zeit/Pause/Bonus, behält Event + Mitarbeiter → schnelles Mehrtages-Anlegen.

### Änderungen an `logHours` + `saveTimesheet`

Beide Funktionen:
- nehmen zusätzlich `breakMinutes` entgegen
- nutzen `calculateHours(..., breakMinutes)`
- prüfen `getDailyHours(uid, date, excludeTsId) + newHours > 10` → `confirm()`-Dialog vor dem `set(...)`

### Validierungen

- `end > start`
- `workHours > 0` nach Pausenabzug, sonst Toast: "Pause länger als Arbeitszeit"
- `tag ∈ eventDateRange(event)`

---

## Feature 3 — Event-Detailansicht

`showEventDetail(id)` (Zeile 1293–1340) wird erweitert. Neue Abschnitte unter den bestehenden Stammdaten:

### Abschnitt: Crew & Kosten

Tabelle aus `timesheetsByEvent(eventId)`, sortiert nach Datum dann Name:

| Datum | Mitarbeiter | Start | Ende | Pause | Arbeit | Bonus | Gesamt | Lohn | Kosten |  |
|---|---|---|---|---|---|---|---|---|---|---|
| 14.04 | Jonas | 18:00 | 23:30 | 30 | 5,00 h | 0,25 | 5,25 h | 14,00 € | 73,50 € | ✎ 🗑 |
| 14.04 | Max | 17:00 | 23:00 | 30 | 5,50 h | 0,00 | 5,50 h | 15,00 € | 82,50 € | ✎ 🗑 |
| 15.04 | Jonas ⚠ | 08:00 | 20:00 | 45 | 11,25 h | 0,25 | 11,50 h | 14,00 € | 161,00 € | ✎ 🗑 |

- ⚠ hinter Name: `isOver10h(uid, date) === true`
- Zeile klickbar → öffnet Timesheet-Modal im Edit-Modus
- ✎ / 🗑 identisch zu aktueller `renderTimesheets`

### Abschnitt: Summen

```
Gesamtstunden:  27,25 h
Gesamtkosten:   381,50 €
Einsatztage:    2
Crew-Size:      2 Personen
Ø Kosten/Tag:   190,75 €
```

### Header-Buttons

- `[PDF Event-Export]` → `exportEvent(eventId)`
- `[+ Zeiterfassung hinzufügen]` → öffnet Admin-Modal vorbefüllt mit `eventId`

### Cross-Link

In `renderTimesheets` (Zeile 1420–1463) werden die Event-Aggregat-Zeilen klickbar: Klick → `showEventDetail(eventId)`.

---

## Feature 4 — Mitarbeiter-Detailansicht

Neues Modal `#user-detail-modal` + Funktion `showUserDetail(uid)`.

### Layout

```
┌────────────────────────────────────────┐
│ Jonas Müller          [Bearbeiten] [X] │
│ jonas@… · 0171… · 14,00 €/h            │
├────────────────────────────────────────┤
│ Zeitraum: [Monat ▾]                    │
│           [von] bis [bis]  [Übernehmen]│
├────────────────────────────────────────┤
│ ▸ Event #1 "Pizza Party Müller"        │
│    14.04   5,25 h    73,50 €           │
│    15.04  11,50 h   161,00 €   ⚠       │
│    ──────────────────                  │
│    Event-Σ  16,75 h  234,50 €          │
│                                        │
│ ▸ Event #2 "Hochzeit Schmidt"          │
│    22.04   8,00 h   112,00 €           │
├────────────────────────────────────────┤
│ Gesamt: 24,75 h  |  346,50 €           │
│ 10h-Verstöße: 1 Tag                    │
└────────────────────────────────────────┘
[PDF Monats-Export]   [PDF Zeitraum-Export]
```

- Default-Zeitraum: aktueller Monat
- Gruppierung: pro Event (Titel + `fmtDate(date)`…`fmtDate(endDate)`)
- Tag-Zeile klickbar → Timesheet-Edit-Modal
- 10h-Verstöße-Zähler: rot, wenn > 0

### Cross-Links

- In `renderTimesheets` User-Aggregat-Zeilen (Zeile 1458–1462) klickbar
- In Settings-Team-Liste (Zeile 2090): Namen-Klick öffnet `showUserDetail`

---

## Feature 5 — 10-Stunden-Warnung

### Stellen

**1) Beim Speichern** (`saveTimesheet`, `logHours`, Event-Card-Handler):

```js
const oldDaily  = getDailyHours(uid, date, editingTsId);
const newDaily  = oldDaily + newEntryTotalHours;
if (newDaily > 10) {
  const ok = confirm(
    `⚠ Warnung: ${userName} käme an ${fmtDate(date)} auf ${newDaily.toFixed(2)} h.\n` +
    `Das überschreitet die gesetzliche Höchstarbeitszeit von 10 h pro Tag.\n\n` +
    `Trotzdem speichern?`
  );
  if (!ok) return;
}
```

**2) In Listen** (`renderTimesheets`, Event-Detail, User-Detail): CSS-Klasse `.over-10h { background: #ffecec; }` auf Zeilen + ⚠-Symbol hinter Name.

**3) Zähler in User-Detail-Summe**: "10h-Verstöße: N Tage" — rot wenn > 0.

**Edit-Case:** Beim Editieren darf der alte Wert nicht doppelt in die Summe fließen → `excludeTsId` als dritter Parameter von `getDailyHours`.

---

## Feature 6 — PDF-Export (jsPDF + autotable)

### Einbindung

```html
<!-- in <head> von index.html -->
<script src="https://cdn.jsdelivr.net/npm/jspdf@2.5.2/dist/jspdf.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/jspdf-autotable@3.8.3/dist/jspdf.plugin.autotable.min.js"></script>
```

Zugriff: `const { jsPDF } = window.jspdf`. Keine weiteren Build-Steps.

### Shared Helpers

```js
function pdfHeader(doc, title, subtitle) {
  // Logo (DataURL const LOGO_PNG = 'data:image/png;base64,...') oben links
  // "Heitmanns Pizza" + Adresse rechts
  // title + subtitle zentriert
  // horizontale Linie
}
function pdfFooter(doc) { /* Seite X / Y + Erstellungsdatum */ }
function pdfCurrency(n) { return n.toLocaleString('de-DE',
  { style: 'currency', currency: 'EUR' }); }
function pdfHours(n)    { return n.toLocaleString('de-DE',
  { minimumFractionDigits: 2, maximumFractionDigits: 2 }) + ' h'; }
function pdfFilename(prefix, parts) {
  return `heitmanns-${prefix}-${parts.join('-')}.pdf`;
}
```

### Vier Export-Funktionen

#### `exportUserMonth(uid, year, month)`

- Titel: `Zeiterfassung {UserName} · {MMMM YYYY}`
- Stammdaten-Kopf: Name, Email, Telefon, Stundenlohn
- `autoTable` Spalten: Datum | Event | Start | Ende | Pause | Arbeit | Bonus | Gesamt | Lohn | Kosten
- Gruppierung: Event-Spalte via `rowSpan`
- Over-10h: `didParseCell` mit `cell.styles.fillColor = [255, 230, 230]`
- Footer-Row: Summe Stunden + Summe Kosten
- Unterschriftenzeile unten: `Mitarbeiter _______  Arbeitgeber _______  Datum _______`
- Filename: `heitmanns-ma-{slug}-{YYYY-MM}.pdf`

#### `exportEvent(eventId)`

- Titel: `Event "{Titel}" · {fmtDate(date)}{–fmtDate(endDate) falls mehrtägig}`
- Stammdaten: Titel, Typ, Ort, Personen, Kunde
- Tabelle gruppiert nach Mitarbeiter, Zwischensumme pro Mitarbeiter, Gesamt
- Filename: `heitmanns-event-{slug}-{YYYY-MM-DD}.pdf`

#### `exportRange({from, to, uid, eventId})`

- Titel: `Zeiterfassung {from}–{to}`
- Subtitle: ggf. Filter ("Mitarbeiter: Jonas" / "Event: Pizza-Party")
- Flache chronologische Tabelle
- Filename: `heitmanns-range-{from}-{to}.pdf`

#### `exportSingleShift(tsId)` (Abwärtskompatibilität)

- Titel: `Einzel-Schicht`
- Eine Zeile als "Quittung"
- `printReceipt(tsId)` (Zeile 1519–1544) delegiert intern an `exportSingleShift`, damit bestehende Aufrufer weiter funktionieren
- Filename: `heitmanns-schicht-{tsId}.pdf`

### Styling

```js
autoTable(doc, {
  theme: 'grid',
  headStyles: { fillColor: [200, 30, 30], textColor: 255 },  // Heitmanns-Rot
  styles: { fontSize: 9, cellPadding: 2 },
  columnStyles: {
    0: { halign: 'center' },    // Datum
    5: { halign: 'right' },     // Arbeit
    9: { halign: 'right' }      // Kosten
  },
  didParseCell(data) {
    if (data.row.raw?.over10h) data.cell.styles.fillColor = [255, 230, 230];
  }
});
```

A4 Portrait, Standard 15mm Rand.

### Aufrufstellen (UI)

- Timesheet-Tab Header: `[PDF Zeitraum-Export]` → öffnet kleines Modal mit from/to/uid?/eventId?
- Event-Detail Header: `[PDF Event-Export]` → `exportEvent(eventId)`
- User-Detail Header: `[PDF Monat]` + `[PDF Zeitraum]`
- Zeilen-Button (bisher 📄): `[PDF]` → `exportSingleShift`

---

## Migration & Abwärtskompatibilität

**Keine Datenmigration nötig.** Alle neuen Felder sind additiv mit sinnvollen Defaults:

| Feld | Fallback |
|---|---|
| `timesheet.breakMinutes` | `?? 0` |
| `user.phone`, `user.address`, `user.birthDate`, `user.hireDate`, `user.notes` | `?? ''` |
| `user.canLogin` | `?? true` |
| `user.createdByAdmin` | `?? false` |

`calculateHours` bleibt abwärtskompatibel durch optionalen 5. Parameter (`breakMinutes = 0`).

`printReceipt` bleibt als Funktion erhalten und delegiert intern an `exportSingleShift` — bestehende `onclick="printReceipt('…')"` funktioniert unverändert.

---

## Teststrategie

Da es **keine Build- oder Test-Infrastruktur** gibt, verläuft das Testing manuell im Browser gegen eine Firebase-Staging-Instanz (oder das Prod-RTDB mit Testdaten, die am Ende gelöscht werden). Checklist:

### Datenmodell & Helpers
- [ ] `calculateHours("18:00","23:30","2026-04-14","2026-04-14", 30)` → 5,00 h
- [ ] `calculateHours("22:00","02:00","2026-04-14","2026-04-15", 0)` → 4,00 h (Übernacht)
- [ ] `eventDateRange({date:"2026-04-14", endDate:"2026-04-16"})` → 3 Einträge
- [ ] `getDailyHours(uid, date, excludeTsId)` zählt bearbeiteten Eintrag nicht doppelt

### Mitarbeiter anlegen
- [ ] Modal öffnet, alle Felder validieren Pflicht
- [ ] Speichern erzeugt `users/u_<ts>` mit `createdByAdmin: true`
- [ ] Admin bleibt eingeloggt (keine Auth-Aktion)
- [ ] Self-Login des neuen Users matcht den DB-Eintrag, alter `u_<ts>`-Datensatz ist danach weg
- [ ] Mailto öffnet mit korrektem Betreff + Body
- [ ] Bearbeiten speichert neue Felder
- [ ] Löschen funktioniert, Self-Delete gesperrt

### Zeiterfassung
- [ ] Einttages-Event zeigt eine Zeile, mehrtägiges Event (endDate+2) zeigt drei Zeilen
- [ ] Dropdowns liefern 15-min-Schritte
- [ ] Speichern erzeugt korrekte Timesheets in Firebase
- [ ] Edit-Mode lädt Werte korrekt vor, Speichern = Update
- [ ] Pause länger als Arbeitszeit → Toast-Fehler, kein Save
- [ ] Admin-Modal "Speichern & nächster Tag" leert Tag/Zeit, behält Event+User
- [ ] Tag-Dropdown zeigt nur Tage des gewählten Events

### Detailansichten
- [ ] Klick auf Event in renderTimesheets → `showEventDetail` mit Kostentabelle
- [ ] Klick auf User in renderTimesheets → `showUserDetail` mit Gruppierung
- [ ] Zeitraum-Filter funktioniert (Monat/Custom)
- [ ] Klick auf Tag-Zeile im User-Detail öffnet Timesheet-Modal im Edit-Modus

### 10h-Warnung
- [ ] Neuer Eintrag, der Daily > 10 bringt → `confirm()` erscheint
- [ ] Abbrechen → Eintrag wird nicht gespeichert
- [ ] OK → Eintrag wird gespeichert, rot in allen Listen
- [ ] Edit eines bestehenden Eintrags zählt nicht doppelt (excludeTsId)
- [ ] User-Detail zeigt Verstoß-Zähler korrekt

### PDF-Export
- [ ] jsPDF lädt aus CDN (Netzwerk-Check)
- [ ] `exportUserMonth` erzeugt PDF, alle Spalten, Gruppierung stimmt, Over-10h rot, Summen richtig
- [ ] `exportEvent` erzeugt PDF mit korrekter Gruppierung pro Mitarbeiter
- [ ] `exportRange` mit und ohne Filter
- [ ] `exportSingleShift` via `printReceipt(tsId)` funktioniert unverändert
- [ ] PDF öffnet sich im Browser, Dateiname korrekt formatiert

### Regression
- [ ] `printReceipt(tsId)` von bestehender Button-Zeile funktioniert weiter
- [ ] Bestehende Timesheets ohne `breakMinutes` werden korrekt mit 0 gerendert
- [ ] Bestehende User ohne neue Felder werden in Settings korrekt ohne Fehler angezeigt
- [ ] Aggregat-Tabellen in `renderTimesheets` zeigen weiterhin korrekte Summen

---

## Offene Punkte / Risiken

- **Firebase Auth Legacy-Users:** User, die vor dem Rework per Self-Login angelegt wurden, haben `createdByAdmin === undefined`. Das Login-Matching prüft explizit `=== true`, also kein Doppel-Match-Risiko.
- **Pausen-Feld nicht in alten Daten:** Rückwirkend nicht problematisch, aber alte Einträge zeigen keine Pause an. Optional: Migration, die pro Event-Typ eine Default-Pause einträgt — **nicht im Scope**, weil User-Entscheidung.
- **jsPDF Logo als DataURL:** Logo muss einmal als Base64 in den Code eingebettet werden (ca. 10–30 kB). Alternative: direkt aus Git laden und per fetch in DataURL wandeln bei Initialisierung.
- **Mobile Usability:** Tages-Listen bei langen Mehrtages-Events (>7 Tage) auf dem Handy — `<details>` collapsible macht es managebar, aber in der ersten Version ist das UX-Limit "paar Tage". Ab 10+ Tagen wäre ein anderes Layout nötig (YAGNI für den Moment).
- **Keine Tests automatisiert:** Manuelles Testing ist die einzige Option bei diesem Stack. Änderungen an `calculateHours` und `getDailyHours` sind besonders sensibel und sollten mit mehreren konkreten Eingaben verifiziert werden (siehe Checklist).

---

## Nächste Schritte

1. Spec vom User reviewen lassen.
2. Mit `writing-plans` einen Implementierungs-Plan schreiben (granulare Tasks, Reihenfolge, Commit-Strategie).
3. Implementierung in `index.html` (ggf. in Feature-Branches pro Feature).
4. Manuelles Testing nach Checklist.
5. Deploy via GitHub Pages (push auf `main`, kein Build-Schritt).
6. Erinnerung: **nach** diesem Rework das Feature "Mitarbeiter-Self-Availability für Events" angehen (bereits als Memory-Notiz hinterlegt).

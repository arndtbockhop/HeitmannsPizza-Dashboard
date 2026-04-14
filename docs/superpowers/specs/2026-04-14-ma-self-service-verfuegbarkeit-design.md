# Mitarbeiter-Self-Service: Verfügbarkeit & Planner-Umbau — Design

**Datum:** 2026-04-14
**Projekt:** HeitmannsPizza-Dashboard
**Scope:** Dedizierter Staff-Tab „Meine Verfügbarkeit" + Planner-Modal als Checkbox-Liste mit Dirty-Tracking

---

## Problem

Mitarbeiter können sich heute bereits einloggen und sich pro Event als verfügbar/nicht-verfügbar melden (`toggleAvail` + `availUIDs`/`unavailUIDs`/`availDetails` sind in `index.html` implementiert). Zwei Probleme:

1. **UI für Staff ist zu versteckt**: Die Buttons „🙋 Habe Zeit" / „🚫 Keine Zeit" hängen am unteren Rand der Event-Karten auf der Termine-Seite. Mitarbeiter finden sie nicht gut, Rückmelde-Quote ist zu niedrig.

2. **Planner für CEO/Admin ist unvollständig**: Die Ansicht zeigt drei getrennte Pools (Verfügbar / Eingeteilt / Weitere). Abgesagte MA sind **komplett unsichtbar** (werden rausgefiltert). „Keine Rückmeldung" ist in einem aufklappbaren `<details>` versteckt. Der CEO sieht nicht auf einen Blick, wer wirklich verfügbar ist und wer abgesagt hat.

## Ziel

- Staff hat einen eigenen, prominenten Tab „Meine Verfügbarkeit" mit großen, touch-freundlichen Ja/Nein-Buttons pro Event.
- Admin/CEO sieht im Planner **alle** Mitarbeiter in einer einzigen Liste, gruppiert nach Status (Zugesagt / Keine Rückmeldung / Abgesagt). Zugesagte sind visuell hervorgehoben, die anderen beiden Gruppen ausgegraut aber trotzdem einteilbar.
- Der Planner warnt klar vor ungespeicherten Änderungen und zeigt den Save-Button prominent oben statt versteckt unten.

## Nicht im Scope

- Kein Backend/DB-Schema-Change: `availUIDs[]`, `unavailUIDs[]`, `availDetails{uid: [dates]}`, `assignedCrew[]` existieren bereits und werden unverändert weiterverwendet.
- Kein neuer Login-Flow: Firebase Auth + PIN funktioniert bereits.
- Keine Änderung an `toggleAvail`, `saveAvailDays`, `savePlanner` (Transaction-Logik bleibt).
- Keine Bereinigung verwaister `availDetails`-Einträge nach Datum-Änderung eines Events.
- Kein Erzwingen von Rückmeldungen (keine Push-Erinnerungen, keine Block-Mechanik).

---

## Architektur

Single-file Vanilla-JS ES-Module in `index.html`. Firebase RTDB Realtime-Listeners verteilen Änderungen live an alle verbundenen Clients. Keine neuen Abhängigkeiten.

**Datenfluss — Staff-View:**
```
Staff klickt "Ich habe Zeit"
  → toggleAvail(eid, true)  [bereits implementiert]
  → runTransaction events/{eid}
  → RTDB pusht an alle Clients
  → events-Listener ruft renderMeineVerfuegbarkeit() + renderPlanner()
```

**Datenfluss — Admin-Planner:**
```
Admin öffnet Planner → plannerOriginalAssigned := [...ev.assignedCrew]
Admin toggled Checkboxen → plannerAssigned mutiert in-memory
  → updatePlannerSaveBar() aktualisiert Counter & Button-State
Admin klickt "Speichern" (oben)
  → savePlanner() [bereits implementiert, keine Änderung nötig]
Admin schließt Modal bei dirty=true
  → Confirm-Dialog, bei "Nein" Modal bleibt offen
```

---

## Komponente 1: Staff-Tab „Meine Verfügbarkeit"

### Sichtbarkeit

- Neuer Sidebar-Eintrag, Mobile-Bottom-Nav-Eintrag.
- CSS-Klasse `.staff-only` (analog zu `.admin-only`/`.superadmin-only`), gesteuert über `body[data-role="staff"]`.
- Admin/Superadmin sehen den Tab **nicht** (sie nutzen den Planner).

### Page-Layout

```
┌────────────────────────────────────────────────┐
│  Meine Verfügbarkeit                           │
│  Trage dich für kommende Events ein            │
│                                                │
│  Du hast für 3 von 7 Events noch keine        │
│  Rückmeldung gegeben.                          │
│                                                │
│  ┌────────────────────────────────────────┐   │
│  │ [Event · Sa 19.04.2026]   [Pending]    │   │
│  │ Hochzeit Müller                         │   │
│  │ 📍 Hamburg, Lindenstraße 3              │   │
│  │                                         │   │
│  │ Deine Rückmeldung:                      │   │
│  │ [ ✅ Ich habe Zeit ] [ ❌ Keine Zeit ] │   │
│  │                                         │   │
│  │ Aktuell: ✅ Zugesagt                    │   │
│  └────────────────────────────────────────┘   │
│  ... weitere Karten ...                        │
└────────────────────────────────────────────────┘
```

### Sortierung

1. Offene Events (weder `availUIDs` noch `unavailUIDs` enthält die eigene UID) zuerst, mit gelbem Rand-Highlight
2. Danach nach Datum aufsteigend
3. Vergangene Events werden ausgeblendet (`date < today()`)

### Button-Verhalten

- **„✅ Ich habe Zeit"**: Ruft `toggleAvail(eid, true)`. Bei mehrtägigen Events öffnet die Funktion bereits das `avail-modal` mit Tag-Checkboxen — unverändert.
- **„❌ Keine Zeit"**: Ruft `toggleAvail(eid, false)`.
- Aktiver Zustand wird farblich hervorgehoben (grün bzw. rot). Beide Buttons sind immer sichtbar, nur einer ist „active".
- Minimum-Touch-Größe 44×44px.

### Status-Zeile

Unterhalb der Buttons zeigt die Karte den aktuellen Zustand:
- `✅ Zugesagt` (einfach-Tages-Event)
- `✅ Zugesagt für 2 von 3 Tagen` (mehrtägig mit Teilauswahl)
- `✅ Zugesagt für alle Tage` (mehrtägig vollständig)
- `❌ Abgesagt`
- `❓ Noch keine Rückmeldung`

### Render-Trigger

- `events`-Listener ruft zusätzlich `renderMeineVerfuegbarkeit()`
- Initial-Render beim ersten `navTo('verfuegbarkeit')`
- Keine eigene Render-Schleife — Live-Sync kommt automatisch über RTDB

---

## Komponente 2: Planner-Umbau (Checkbox-Liste)

### Modal-Layout (neu)

```
┌──────────────────────────────────────────────────┐
│  Personal für: Hochzeit Müller            [ X ] │
│  Sa 19.04.2026 · Hamburg                        │
│                                                  │
│  ┌────────────────────────────────────────────┐ │ ← sticky bar
│  │ [ 💾 Speichern (3 Änderungen) ]            │ │
│  │ 3 zugesagt · 1 abgesagt · 2 ohne Antwort  │ │
│  │ Eingeteilt: 2 MA                           │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
│  ZUGESAGT (3)                                   │
│  ☑ ✅ Anna Schmidt            eingeteilt        │
│  ☑ ✅ Ben Weber               eingeteilt        │
│  ☐ ✅ Clara Meyer                               │
│                                                  │
│  KEINE RÜCKMELDUNG (2)                          │
│  ☐ ❓ David Klein           (ausgegraut)        │
│  ☐ ❓ Emma Lang             (ausgegraut)        │
│                                                  │
│  ABGESAGT (1)                                   │
│  ☐ ❌ Felix Braun           (ausgegraut)        │
└──────────────────────────────────────────────────┘
```

### Gruppierung & Sortierung

Eine einzige Liste, drei sichtbare Abschnitte in dieser Reihenfolge:

1. **Zugesagt** — UIDs aus `availUIDs`. Volle Opacity, grüner Status-Pill.
2. **Keine Rückmeldung** — `state.users` minus `availUIDs` minus `unavailUIDs` minus superadmin-Rolle. Opacity 0.5.
3. **Abgesagt** — UIDs aus `unavailUIDs`. Opacity 0.5, rotes ❌-Icon.

Innerhalb jeder Gruppe alphabetisch nach `u.name`.

### Checkbox-Verhalten

- Jede Zeile beginnt mit einer Checkbox. Zustand = `plannerAssigned.includes(uid)`.
- Klick auf Checkbox oder Row-Body toggled den Eintrag in `plannerAssigned`.
- Nach jedem Toggle: `updatePlannerSaveBar()` neu rechnen.
- **Abgesagte anhaken**: Warn-Toast bei jedem Toggle auf einen Abgesagten: _„Felix Braun hat abgesagt — trotzdem einteilen?"_ (nicht blockierend, Toast verschwindet nach 3s). Eintrag wird trotzdem gesetzt. Kein Session-State nötig, einfache Implementierung.
- **Kein Toast** bei „Keine Rückmeldung" — nur visuell ausgegraut.

### Warnung: Eingeteilt & Abgesagt

Wenn `plannerAssigned.includes(uid) && unavailUIDs.includes(uid)` → Row bekommt zusätzlich einen roten linken Rand (`border-left: 3px solid red`) und Tooltip `title="Eingeteilt, hat aber abgesagt"`. Macht den Konflikt sofort sichtbar.

### Sticky Save-Bar

Oberhalb der Liste, unterhalb des Modal-Titels, **position: sticky; top: 0** innerhalb der scrollbaren Modal-Body-Container:

- **Button „💾 Speichern (N Änderungen)"**: Text zeigt Live-Counter. `N = 0` → Button ist disabled (grau) und Text: „Keine Änderungen".
- **Zweite Zeile**: Aggregierte Counts (Zugesagt / Abgesagt / Keine Rückmeldung / Eingeteilt).

Counter-Berechnung:
```js
function plannerChangeCount() {
    const a = new Set(plannerOriginalAssigned);
    const b = new Set(plannerAssigned);
    let count = 0;
    a.forEach(uid => { if (!b.has(uid)) count++; });
    b.forEach(uid => { if (!a.has(uid)) count++; });
    return count;
}
```

### Dirty-Tracking & Close

- Beim `openPlannerModal(id)`: `plannerOriginalAssigned = [...(ev.assignedCrew || [])]`.
- `isPlannerDirty()` vergleicht `plannerOriginalAssigned` vs `plannerAssigned` (als Sets).
- **Neuer Close-Handler `plannerClose()`**:
  - wenn `!isPlannerDirty()` → direkt `closeModal('planner-modal')`
  - wenn dirty → `if (confirm('Du hast ungespeicherte Änderungen. Trotzdem schließen?')) closeModal('planner-modal')`
- Trigger für `plannerClose()`: X-Button, Overlay-Klick, Escape-Taste.
- Nach erfolgreichem `savePlanner()` → `plannerOriginalAssigned = [...plannerAssigned]`, Counter zurück auf 0. (Momentan schließt savePlanner das Modal direkt; wir passen an, damit die Bar sauber zurückspringt — beide Pfade werden im Plan abgedeckt.)

### Footer-Entfall

Der bestehende Modal-Footer mit „Abbrechen" / „Speichern" fällt weg. Alles läuft über die Sticky-Bar oben.

---

## Dateistruktur

Ein-Datei-Projekt `index.html`. Alle Änderungen in dieser Datei:

| Bereich | Aktion | Ungefähre Zeilen |
|---|---|---|
| Sidebar Nav-Item | Einfügen `staff-only` Button | ~582 |
| Mobile-Bottom-Nav | Einfügen `staff-only` Button | ~783 |
| Neue Page `page-verfuegbarkeit` | Einfügen nach `page-dashboard` | ~728 |
| `planner-modal` HTML | Umbau: Sticky-Bar, Single-Container `#planner-list`, Footer weg | 788-810 |
| CSS `.staff-only` | Neue Regel analog `.admin-only` | ~470 |
| CSS Planner-Gruppen | Neue Klassen: `.planner-group-header`, `.planner-row`, `.planner-row.is-faded`, `.planner-row.is-conflict`, `.planner-sticky-bar` | neuer Block |
| CSS Staff-Karten | Große Touch-Buttons, Card-Styling | neuer Block |
| JS Neue Funktionen | `renderMeineVerfuegbarkeit`, `updatePlannerSaveBar`, `plannerClose`, `togglePlannerCheckbox`, `plannerChangeCount`, `isPlannerDirty` | ~2140 und ~2250 |
| JS Umbauten | `renderPlannerUI`, `openPlannerModal`, `savePlanner` (minimal) | 2246-2303 |
| JS Löschen | `plannerMove` (durch Toggle ersetzt) | 2275-2279 |
| Events-Listener | Zusätzlich `renderMeineVerfuegbarkeit()` aufrufen | ~1861 |
| Window-Export | `plannerClose`, `togglePlannerCheckbox` ergänzen; `plannerMove` entfernen | ~3703 |

---

## Edge Cases

**Mehrtägige Events**: Bestehende `avail-modal`-Flow bleibt. Status-Zeile in Staff-Karte zeigt Teilauswahl (z.B. „Zugesagt für 2 von 3 Tagen"), berechnet aus `ev.availDetails[uid].length` vs Event-Dauer.

**Staff meldet sich nach Einteilung ab**: `toggleAvail` entfernt aus `availUIDs`, fügt `unavailUIDs` hinzu, **berührt `assignedCrew` nicht**. Planner-Row zeigt dann Konflikt-Highlight (roter linker Rand + Tooltip). CEO muss manuell entscheiden.

**Event-Datum wird nach Zusage verschoben**: `availDetails[uid]` kann veraltete Datums-Strings enthalten. Nicht im Scope — in bestehender Code-Basis bereits so.

**Event gelöscht**: `assignedCrew` verschwindet mit dem Event. Kein zusätzlicher Cleanup nötig.

**Concurrent Edit zweier Admins**: `savePlanner` nutzt `runTransaction` für `assignedCrew`. Zweiter Save überschreibt ersten. Akzeptabel für dieses Einzel-Tenant-Setup.

**Staff mit `canLogin: false`**: Kann sich nicht einloggen, sieht also auch den neuen Tab nicht. Keine Sonderbehandlung nötig.

---

## Testplan (manuell)

1. Staff-Login → Sidebar zeigt `Meine Verfügbarkeit`, Admin/Superadmin sieht ihn nicht
2. Staff klickt „Ich habe Zeit" auf Einzeltag-Event → Status-Zeile wechselt auf „✅ Zugesagt", grüner Button aktiv
3. Staff klickt „Keine Zeit" → Status auf „❌ Abgesagt"
4. Staff klickt „Ich habe Zeit" auf mehrtägigem Event → `avail-modal` öffnet mit Tag-Checkboxen, Auswahl speichert, Karte zeigt „Zugesagt für 2 von 3 Tagen"
5. Info-Zeile oben zeigt korrekten Count offener Events
6. Sortierung: offene Events erscheinen oben
7. Admin öffnet Planner → eine Liste, drei Gruppen sichtbar, korrekte Opacity
8. Admin tickt Zugesagten an → Counter `(1 Änderung)`, Save-Button aktiv
9. Admin tickt Abgesagten an → Warn-Toast erscheint, Eintrag gesetzt, Counter `(2 Änderungen)`
10. Admin untickt wieder → Counter `(1 Änderung)` oder `0` falls nur die eine Änderung übrig
11. Admin klickt `X` bei dirty → Confirm-Dialog, bei „Nein" Modal bleibt, bei „Ja" Modal schließt ohne Save
12. Admin klickt Save → Counter `0`, Button disabled, `assignedCrew` in DB aktualisiert, Notifications an neue Zusagen gesendet (bestehende Logik)
13. Staff (Browser-Tab 2) meldet sich nach Einteilung ab → Planner-Row im Browser-Tab 1 bekommt roten Rand + Tooltip
14. Planner mit null MA-Datensätzen → leere Gruppen zeigen keine Header, Save-Bar funktioniert
15. Planner ohne Zusagen und Absagen → nur „Keine Rückmeldung"-Gruppe sichtbar, Warn-Toast wird nie getriggert

---

## Offene Punkte für die spätere Plan-Phase

- Exakte CSS-Werte für Opacity/Abstände → beim Implementieren passend zum bestehenden Apple-Style einstellen (Orientierung: `.is-faded` = Opacity 0.55)
- `savePlanner` schließt das Modal weiterhin automatisch (konsistent zum aktuellen Verhalten). Vor dem Schließen wird `plannerOriginalAssigned := [...plannerAssigned]` gesetzt, sodass der Dirty-Check beim nächsten Öffnen sauber startet.

# PDF-, Pausen- & Auswertung-Verbesserungen — Design

**Datum:** 2026-04-15
**Projekt:** HeitmannsPizza-Dashboard
**Scope:** Drei zusammenhängende Verbesserungen nach User-Feedback auf dem Zeiterfassungs-Rework

---

## Problem

Nach dem Zeiterfassungs-Rework (2026-04-13) sind drei Punkte aufgefallen:

1. **Quittung-PDF (`exportSingleShift`):** Die Signatur-Zeile ist aktuell über die gesamte Seitenbreite verteilt (Aushilfe links x=15, Ort Mitte x=80, Datum rechts x=145). Das wirkt ausgefranst. Der Kunde möchte ein kompakteres, links-gruppiertes Layout.

2. **Gesetzliche Pausenregelung (§4 ArbZG):** Im ersten Rework wurde bewusst auf eine Pausen-Automation verzichtet. In der Praxis vergessen Admins jetzt die Pause einzutragen — es kommt zu Einträgen mit 8 h Arbeit ohne Pause. Eine weiche Automation soll das verhindern, ohne dem Admin die manuelle Kontrolle zu nehmen.

3. **Event-PDF (`exportEvent`):** Der aktuelle Flat-Table-Export listet jede Schicht einzeln in einer Sammeltabelle mit einer Zwischensumme pro MA. Layout unterscheidet sich vom `exportUserMonth`-PDF und vom Quittungs-PDF. Bei mehrtägigen Events erscheint derselbe MA in mehreren Zeilen ohne klare visuelle Gruppierung. Der Kunde möchte ein einheitliches, Block-basiertes Layout analog zum Monats-PDF — jeder MA als eigener, deutlich abgesetzter Block mit alphabetischer Sortierung.

## Ziel

- Quittung-PDF: Beide Signatur-Elemente (Ort/Datum-Zeile + Aushilfe-Unterschrift) auf der linken Seitenhälfte, vertikal gestapelt.
- Zeiterfassungs-Modal: Bei Eingabe von Start/Ende wird die gesetzliche Mindestpause automatisch vorgeschlagen. Admin kann überschreiben. Banner auf bestehenden Einträgen ohne regelkonforme Pause.
- Event-PDF: Ein Block pro MA — Header mit Name + Stundenlohn, Tabelle mit allen Schichten des MAs, MA-Zwischensumme. Event-Summe am Ende. Signatur-Block + Disclaimer. MAs alphabetisch nach Name.

## Nicht im Scope

- Keine harte Blockade: Admin kann Pausen weiter manuell auf 0 stellen, falls wirklich nicht gepaust wurde.
- Keine Änderung an `exportUserMonth` oder `exportRange` — die funktionieren bereits.
- Kein Refactoring der Daten-Struktur von `timesheets`.
- Keine Änderung an der Break-Option-Liste (`BREAK_OPTIONS`).

---

## Architektur

Single-file Vanilla-JS-ES-Module in `index.html`. Alle Änderungen sind lokal in drei Funktionen und einer neuen Hilfsfunktion. Keine neuen Dependencies.

**Datenfluss — Pausen-Auto-Vorschlag:**
```
Admin wählt Start/Ende im Timesheet-Modal
  → updateTimesheetLivePreview() rechnet diff (bestehend)
  → NEUE Logik: suggestedBreak(diffH) → 0 | 30 | 45
  → wenn Admin die Pause noch nicht manuell gesetzt hat
     → $('ts-break').value = suggestedBreak
     → Live-Preview rendert mit neuer Pause
  → Admin kann manuell überschreiben (Flag merken)
```

**Datenfluss — Quittung-PDF:**
```
printReceipt(tsId) → exportSingleShift(tsId) [bestehend]
  → pdfSignatureBlock(doc, y, { leftLabel: 'Aushilfe', withOrt: true }) [geändert]
    → zeichnet linke Spalte: Ort-Bar, Datum-Bar, Aushilfe-Bar
```

**Datenfluss — Event-PDF:**
```
exportEvent(eventId) [umgebaut]
  → gruppiert timesheets nach userId, sortiert alphabetisch nach user.name
  → für jeden MA: pdfHeader-Block + autoTable + Zwischensumme-Zeile
  → Event-Summe als eigene Footer-Zeile am Ende
  → pdfSignatureBlock mit Disclaimer (links-Layout)
```

---

## Komponente A — Quittung-PDF: Links-Layout

### Aktuelles Layout (x-Koordinaten in mm)

```
Disclaimer (15-195, über volle Breite)

_______________   _______________   _______________
Aushilfe          Ort               Datum
(x=15)            (x=80)            (x=145)
```

### Neues Layout

```
Disclaimer (15-195, über volle Breite — unverändert)

__________   __________
Ort          Datum
(x=15)       (x=55)

_________________________
Aushilfe
(x=15, Länge wie vorher)
```

Beide Blöcke linksbündig auf der ersten Seitenhälfte. Ort/Datum-Zeile mit zwei kurzen Bars (je ~35 mm lang). Darunter leerer Abstand (~10 mm), dann volle Aushilfe-Bar.

### Änderung in `pdfSignatureBlock`

Die Funktion wird erweitert, aber ihre Signatur bleibt rückwärtskompatibel. Die `withOrt`-Variante zeichnet neu:

1. Disclaimer wie bisher (wenn `withDisclaimer`)
2. `y += lines.length * 5 + 12`
3. Ort-Bar: `doc.text('_______________', 15, y); doc.text('Ort', 15, y+5);`
4. Datum-Bar: `doc.text('_______________', 55, y); doc.text('Datum', 55, y+5);`
5. `y += 18`
6. Aushilfe-Bar: `doc.text('_______________________', 15, y); doc.text('Aushilfe', 15, y+5);`

Der `needed`-Höhen-Check muss auf ~70 mm erhöht werden, damit der Block nicht halb auf der nächsten Seite landet.

**Ohne `withOrt` (Monats-PDF `exportUserMonth`, Range-PDF `exportRange`):** Keine Änderung — der einfache Mitarbeiter+Datum-Block bleibt nebeneinander wie heute. Der Kunde hat nur die Quittung kritisiert.

---

## Komponente B — Gesetzliche Pausenregelung (Auto-Vorschlag + Retro-Banner)

### Regel

Abgeleitet aus §4 ArbZG:
- Arbeitszeit 0 – 6 h: keine Pause verpflichtend → 0 min
- Arbeitszeit > 6 h bis ≤ 9 h: 30 min verpflichtend
- Arbeitszeit > 9 h: 45 min verpflichtend

Die Berechnung bezieht sich auf die **Brutto-Arbeitszeit zwischen Start und Ende** (nicht auf die Netto-Arbeitszeit nach Abzug der Pause, da sonst Zirkelschluss).

### Hilfsfunktion `suggestedBreakMinutes(startStr, endStr, startDate, endDate)`

Liefert die gesetzliche Mindestpause in Minuten.

```js
function suggestedBreakMinutes(startStr, endStr, startDate, endDate) {
    if (!startStr || !endStr) return 0;
    const gross = calculateHours(startStr, endStr, startDate, endDate, 0);
    if (gross > 9) return 45;
    if (gross > 6) return 30;
    return 0;
}
```

Nutzt die bestehende `calculateHours`, pausiert mit 0.

### Integration ins Timesheet-Modal

Im Modal gibt es ein Dirty-Flag `tsBreakManuallyTouched` als Session-State:

- Bei `openTimesheetModal(id)`:
  - wenn `ts` (Bearbeitungsmodus): `tsBreakManuallyTouched = true` — bestehende Werte nicht überschreiben
  - wenn neu: `tsBreakManuallyTouched = false`
- Im `onchange` des `#ts-break`-Selects: `tsBreakManuallyTouched = true`
- In `updateTimesheetLivePreview`:
  - wenn `!tsBreakManuallyTouched`: `$('ts-break').value = suggestedBreakMinutes(...)` anwenden, **bevor** die Preview-Berechnung läuft
  - dann wie bisher `brk = Number($('ts-break')?.value || 0)` lesen

Dadurch füllt sich das Pause-Select live beim Ändern von Start/Ende, solange der Admin es nicht angefasst hat. Sobald der Admin die Pause einmal setzt, wird sie nicht mehr überschrieben.

**Edge Case:** Wenn Admin die Pause auf 0 setzt (manueller Override), bleibt sie auf 0 auch wenn Arbeitszeit > 6 h ist. Das ist gewollt — manchmal gab es objektiv keine Pause, und das muss dokumentierbar bleiben.

### Retro-Banner auf bestehenden Einträgen

Bestehende Einträge ohne regelkonforme Pause sollen **sichtbar** markiert werden, aber nicht automatisch geändert werden (Admin-Hoheit).

**Umsetzung in der Timesheet-Liste (bestehende `renderTimesheets`-/Event-Detail-Tabelle):**
- Hilfs-Check: `missingLegalBreak(ts)` → gibt `true`, wenn `ts.breakMinutes < suggestedBreakMinutes(ts.start, ts.end, ts.date, null)`
- Zeilen mit `missingLegalBreak === true` bekommen zusätzlich Warn-Icon in der Pause-Spalte: `⚠️ X min` + CSS `title="Gesetzliche Mindestpause: Y min"`
- Im Event-Detail und im Timesheet-Übersichts-Table.

**Kein Banner in den PDFs** — der PDF-Export ist ein Snapshot, Warnings würden die Steuerberater-Sicht verwirren. Die Warnung ist nur im UI sichtbar.

---

## Komponente C — Event-PDF: Block-Layout

### Neues Layout (statt flacher Tabelle)

```
[PDF-Header: "Event: Hochzeit Müller" / "Sa 19.04.2026 – So 20.04.2026"]
Typ: Hochzeit
Ort: Hamburg, Lindenstraße 3
Personen: 120

─────────────────────────────────────────────────
Anna Schmidt · 12,50 €/h
┌──────────┬─────────────┬──────┬───────┬────────┬──────────┐
│ Datum    │ Zeit        │ Pause│ Arbeit│ Gesamt │ Kosten   │
├──────────┼─────────────┼──────┼───────┼────────┼──────────┤
│ 19.04.   │ 16:00–23:00 │ 30min│ 6.50h │ 6.50h  │ 81,25 €  │
│ 20.04.   │ 14:00–20:00 │ 30min│ 5.50h │ 5.50h  │ 68,75 €  │
├──────────┴─────────────┴──────┴───────┼────────┼──────────┤
│                     Summe Anna Schmidt│12.00 h │150,00 €  │
└───────────────────────────────────────┴────────┴──────────┘

─────────────────────────────────────────────────
Ben Weber · 11,00 €/h
┌──────────┬─────────────┬──────┬───────┬────────┬──────────┐
│ Datum    │ Zeit        │ Pause│ Arbeit│ Gesamt │ Kosten   │
├──────────┼─────────────┼──────┼───────┼────────┼──────────┤
│ 19.04.   │ 16:00–00:00 │ 30min│ 7.50h │ 7.50h  │ 82,50 €  │
├──────────┴─────────────┴──────┴───────┼────────┼──────────┤
│                      Summe Ben Weber  │ 7.50 h │ 82,50 €  │
└───────────────────────────────────────┴────────┴──────────┘

─────────────────────────────────────────────────

          ┌─────────────────────────────┬────────┬──────────┐
          │                 Event-Summe │19.50 h │232,50 €  │
          └─────────────────────────────┴────────┴──────────┘

[Disclaimer-Text zur Aushilfe]

_________   _________
Ort         Datum

_________________________
Aushilfe
```

### Umsetzungs-Schritte in `exportEvent`

1. `list = timesheetsByEvent(eventId)` (bestehend)
2. Gruppierung: `byUser = {}` mit `{ [uid]: ts[] }` (bestehend)
3. **Neu:** Sortierung der MA-Blöcke: `Object.entries(byUser).sort((a,b) => getUserName(a[0]).localeCompare(getUserName(b[0]), 'de'))`
4. Innerhalb jedes Blocks: Sortierung der Schichten nach `date` aufsteigend
5. Pro MA: `currentY` updaten, Name + Wage-Zeile zeichnen, dann `doc.autoTable({ startY: currentY+3, head: [[...]], body, foot: [['Summe ${name}', ...]] })`
6. Nach `autoTable`: `currentY = doc.lastAutoTable.finalY + 8` (Abstand zum nächsten Block)
7. Seitenumbruch-Behandlung kommt automatisch von `autoTable` (nur ein Seitenumbruch mitten im Block möglich — akzeptabel)
8. Nach allen MA-Blöcken: eigene kleine Event-Summen-Tabelle ohne Head, nur Body+Foot oder einfach ein zweites `autoTable` mit nur einer foot-Zeile
9. Signatur-Block via `pdfSignatureBlock(doc, currentY + 15, { withDisclaimer: true, leftLabel: 'Aushilfe', withOrt: true })` — nutzt bereits das neue Links-Layout aus Komponente A.

### Pro-MA-Header-Block (Name + Wage)

Nicht als autoTable-Head-Row, sondern als eigener `doc.text`-Block oberhalb der Tabelle — so trennt er sich visuell ab. Zwei Zeilen:
- Zeile 1 (bold, 11 pt): `Anna Schmidt`
- Zeile 2 (normal, 9 pt, grau): `Stundenlohn: 12,50 €`

Darunter beginnt `autoTable` mit den 6 Spalten `Datum | Zeit | Pause | Arbeit | Gesamt | Kosten`.

### MA-Zwischensumme in der Tabelle

Als `foot`-Zeile pro `autoTable`:
```js
foot: [[
    { content: `Summe ${u.name}`, colSpan: 4, styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
    { content: pdfHours(subH), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
    { content: pdfCurrency(subC), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } }
]]
```

### Event-Summe am Ende

Entweder als eigenes Mini-`autoTable` ohne Head:
```js
doc.autoTable({
    startY: currentY + 8,
    margin: { bottom: 25 },
    body: [[
        { content: 'Event-Summe', colSpan: 4, styles: { halign: 'right', fontStyle: 'bold', fillColor: [0,0,0], textColor: 255 } },
        { content: pdfHours(totalH), styles: { halign: 'right', fontStyle: 'bold', fillColor: [0,0,0], textColor: 255 } },
        { content: pdfCurrency(totalC), styles: { halign: 'right', fontStyle: 'bold', fillColor: [0,0,0], textColor: 255 } }
    ]],
    theme: 'grid',
    styles: PDF_STYLES
});
```

(Schwarzer Hintergrund = visuelle Gleichheit zum `exportUserMonth`-Footer.)

---

## Dateistruktur

Ein-Datei-Projekt `index.html`. Alle Änderungen in dieser Datei:

| Bereich | Aktion | Ungefähre Zeilen |
|---|---|---|
| `pdfSignatureBlock` | Logik im `withOrt`-Zweig auf Links-Layout umbauen, `needed` auf 70 mm setzen | 1511-1542 |
| `suggestedBreakMinutes` | Neue Hilfsfunktion direkt nach `calculateHours` | ~1382 |
| `missingLegalBreak` | Neue Hilfsfunktion direkt nach `suggestedBreakMinutes` | ~1395 |
| `openTimesheetModal` | `tsBreakManuallyTouched` initialisieren | 2988-3019 |
| `updateTimesheetLivePreview` | Auto-Fill-Logik vor der bestehenden Berechnung | 3031-3059 |
| `$('ts-break').onchange` | Flag setzen zusätzlich zum bestehenden Preview-Hook | ~3003 |
| Timesheet-Listen-Renderer | Warn-Icon in Pause-Spalte bei `missingLegalBreak` | ~2750 und ~3959 |
| `exportEvent` | Komplettumbau zum Block-Layout | 1634-1704 |

---

## Edge Cases

**Event mit 0 Schichten:** `exportEvent` zeigt Header + Event-Summe = 0. Kein Block, kein Crash. Warn-Text „Keine Schichten erfasst." statt Tabellen-Bereich.

**MA in der DB gelöscht:** `find(state.users, uid, 'uid')` liefert `null`. Fallback-Name „Gelöschter Mitarbeiter". Wage = `DEFAULT_WAGE`. Sortierung mit diesem Fallback-Namen.

**Pause-Auto-Vorschlag bei mehrtägigem Event mit Mitternachts-Übergang:** `calculateHours` kennt `endDateStr`. `suggestedBreakMinutes` erhält beide Datums, damit wie 22:00–06:00 korrekt als 8 h gerechnet wird (→ 30 min).

**Retro-Warning auf sehr alte Einträge:** Der Warn-Icon erscheint auch rückwirkend. Das ist erwünscht — Admin soll wissen, welche Einträge problematisch sind. Kein automatisches Update (würde historische Daten verändern, nicht akzeptabel für Lohnbuchhaltung).

**Pause = 0 explizit gewünscht:** Admin kann nach dem Auto-Fill manuell auf 0 zurückstellen. Das Dirty-Flag bleibt dann `true`, Warn-Icon in der Liste erscheint trotzdem (gewollt als Doku-Signal).

**Seitenumbruch mitten im MA-Block:** `autoTable` bricht sauber in Tabellen-Zeilen um. Wenn der Header (Name + Wage) auf Seite 1 steht und die Tabelle auf Seite 2 beginnt, ist das suboptimal. **Mitigation:** Vor dem MA-Block-Header prüfen, ob mindestens 40 mm Platz verbleiben (`pageH - currentY > 40`), sonst `doc.addPage(); currentY = 30` vor dem Header.

**MA mit Schichten aus mehreren Tagen desselben Events:** Die Schichten werden im Block untereinander aufgelistet (sortiert nach Datum), die Zwischensumme summiert alle → User-Requirement erfüllt.

---

## Testplan (manuell)

**A — Quittung-PDF:**
1. Schicht mit beliebigem Ort erfassen → „Quittung" klicken → PDF öffnet
2. Signatur-Bereich: Disclaimer oben, darunter Ort-Bar + Datum-Bar nebeneinander links, darunter Aushilfe-Bar links, alle auf linker Seitenhälfte
3. Seitenumbruch: Schicht mit sehr langem Disclaimer-Text → Signatur landet auf neuer Seite komplett, nicht zerrissen
4. Monats-PDF (`exportUserMonth`) öffnen → unverändertes Layout (Mitarbeiter links + Datum rechts, ohne Ort)

**B — Pausen-Auto:**
5. Neuer Timesheet-Eintrag: Start 16:00, Ende 23:00, Pause-Select zeigt automatisch 30 min, Preview rechnet mit 30 min Pause
6. Start auf 14:00, Ende 00:00 → Pause springt auf 45 min
7. Start auf 18:00, Ende 22:00 → Pause auf 0 min
8. Admin stellt Pause manuell auf 45 min → wechselt Start auf 18:00, Ende 22:00 → Pause bleibt auf 45 (nicht überschreiben)
9. Bearbeitung eines bestehenden Eintrags → Pause-Wert bleibt wie gespeichert, wird nicht überschrieben
10. Timesheet-Tabelle: bestehender Eintrag mit 8 h Arbeit und 0 min Pause → Warn-Icon `⚠️ 0 min` mit Tooltip
11. Event-Detail: gleiche Warn-Ikone in der Pause-Spalte

**C — Event-PDF:**
12. Event mit 3 MAs × 1 Schicht → 3 Blöcke, alphabetisch nach Name sortiert
13. Event mit 1 MA × 2 Schichten (mehrtägig) → 1 Block, 2 Zeilen, MA-Summe korrekt
14. Event mit 0 Schichten → Header + Warn-Text, keine Crashs
15. Event mit 8 MAs (mehrere Seiten) → Seitenumbruch zwischen Blöcken, kein Block zerrissen mit Header auf der vorigen Seite
16. Event-Summe unten in schwarzem Block, Summen korrekt
17. Signatur-Block mit Disclaimer im neuen Links-Layout am Ende
18. MA in DB gelöscht (Fake: userId zeigt auf nicht existenten User) → Block „Gelöschter Mitarbeiter", kein Crash

---

## Offene Punkte für die Plan-Phase

- Exakte y-Abstände zwischen Ort/Datum-Bar und Aushilfe-Bar (11 mm vs. 18 mm) → beim Implementieren mit dem tatsächlichen PDF vergleichen, aktuell 18 mm als Vorschlag
- Falls `doc.autoTable` in manchen Edge-Fällen kein `finalY` setzt (z. B. bei leerem Body), muss `currentY` manuell getrackt werden — vermutlich nicht relevant, aber im Plan als Sicherungsnetz einbauen
- Warn-Icon-Style: `⚠️` als Unicode-Emoji vs. HTML-Entity — existierender Code nutzt schon `⚠️` in Event-Cards, also konsistent bleiben

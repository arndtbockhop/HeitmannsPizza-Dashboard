# PDF-, Pausen- & Auswertung-Verbesserungen — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Quittungs-PDF auf Links-Layout umbauen, gesetzliche Pause als Auto-Vorschlag im Timesheet-Modal inkl. Retro-Warn-Icon, Event-PDF als Block-Layout pro Mitarbeiter.

**Architecture:** Alle Änderungen in `index.html` (Single-file vanilla JS ES-Module). Keine neuen Dateien, keine neuen Dependencies. Verifikation via `/tmp/claude/syncheck.cjs` (JS-Syntax) + manueller Testrunde im Browser.

**Tech Stack:** Vanilla JS, Firebase RTDB, jsPDF 2.5.2 + jspdf-autotable 3.8.3.

**Spec:** `docs/superpowers/specs/2026-04-15-pdf-pausen-auswertung-improvements-design.md`

**Security-Hook-Hinweis:** Der PreToolUse-Hook blockiert das literale Muster `el.inner` gefolgt von `HTML =` in neu geschriebenem Code. Wenn du ein neues derartiges Assignment einbauen musst, nutze die Bracket-Notation `el[['inner','HTML'].join('')] = html`. Alle Edits in diesem Plan modifizieren nur bestehende Template-Literals innerhalb bereits existierender Zuweisungen — du fügst keine neuen hinzu.

---

## Task 1: Hilfsfunktion `suggestedBreakMinutes` + `missingLegalBreak`

**Files:**
- Modify: `index.html` (nach `calculateHours`, ~Zeile 1381)

- [ ] **Step 1: Einfügen direkt nach `calculateHours`**

Füge nach Zeile 1381 (`return net > 0 ? net : 0;` + schließende `}`) und vor Zeile 1383 (`// TIMESHEET CONSTANTS AND HELPERS`-Kommentar) ein:

```js
function suggestedBreakMinutes(startStr, endStr, startDateStr, endDateStr) {
    if (!startStr || !endStr) return 0;
    const gross = calculateHours(startStr, endStr, startDateStr, endDateStr, 0);
    if (gross > 9) return 45;
    if (gross > 6) return 30;
    return 0;
}

function missingLegalBreak(ts) {
    if (!ts || !ts.start || !ts.end) return false;
    const required = suggestedBreakMinutes(ts.start, ts.end, ts.date, null);
    return Number(ts.breakMinutes || 0) < required;
}
```

- [ ] **Step 2: Syntax-Check**

```bash
node /tmp/claude/syncheck.cjs
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add suggestedBreakMinutes and missingLegalBreak helpers"
```

---

## Task 2: Timesheet-Modal — Auto-Fill-Logik mit Dirty-Flag

**Files:**
- Modify: `index.html` `openTimesheetModal` (Zeilen 2988–3019)
- Modify: `index.html` `updateTimesheetLivePreview` (Zeilen 3031–3059)
- Modify: `index.html` `saveTimesheet` (Reset-Pfad, ~Zeile 3108)

- [ ] **Step 1: Modul-State-Variable vor `openTimesheetModal` hinzufügen**

Suche Zeile 2988 `function openTimesheetModal(id) {` und füge direkt davor ein:

```js
let tsBreakManuallyTouched = false;
```

- [ ] **Step 2: `openTimesheetModal` erweitern — Flag initialisieren**

Aktueller Stand Zeile 2988–2992:

```js
function openTimesheetModal(id) {
    _ensureTimesheetLists();
    state.editTimesheetId = id || null;
    const ts = id ? find(state.timesheets, id) : null;
```

Ersetze durch:

```js
function openTimesheetModal(id) {
    _ensureTimesheetLists();
    state.editTimesheetId = id || null;
    const ts = id ? find(state.timesheets, id) : null;
    tsBreakManuallyTouched = !!ts;
```

(Im Bearbeiten-Modus wird der Wert als „manuell gesetzt" behandelt, damit bestehende Einträge nicht überschrieben werden.)

- [ ] **Step 3: Break-Select-change-Hook erweitern**

Suche Zeilen 3003–3006:

```js
    ['ts-start','ts-end','ts-break','ts-bonus','ts-user-id','ts-date'].forEach(eid => {
        const el = $(eid);
        if (el) el.onchange = updateTimesheetLivePreview;
    });
```

Ersetze durch:

```js
    ['ts-start','ts-end','ts-break','ts-bonus','ts-user-id','ts-date'].forEach(eid => {
        const el = $(eid);
        if (el) el.onchange = updateTimesheetLivePreview;
    });
    const breakEl = $('ts-break');
    if (breakEl) {
        breakEl.addEventListener('change', () => { tsBreakManuallyTouched = true; });
    }
```

- [ ] **Step 4: `updateTimesheetLivePreview` — Auto-Fill vor Preview-Berechnung**

Suche Zeilen 3031–3040:

```js
function updateTimesheetLivePreview() {
    const start = $('ts-start')?.value, end = $('ts-end')?.value;
    const brk = Number($('ts-break')?.value || 0);
    const bonus = Number($('ts-bonus')?.value || 0);
    const uid = $('ts-user-id')?.value;
    const date = $('ts-date')?.value;
    const user = find(state.users, uid, 'uid');
    const wage = user ? Number(user.wage) || DEFAULT_WAGE : DEFAULT_WAGE;
    const work = calculateHours(start, end, date, null, brk);
    const total = work + bonus;
```

Ersetze durch:

```js
function updateTimesheetLivePreview() {
    const start = $('ts-start')?.value, end = $('ts-end')?.value;
    const bonus = Number($('ts-bonus')?.value || 0);
    const uid = $('ts-user-id')?.value;
    const date = $('ts-date')?.value;
    if (!tsBreakManuallyTouched && start && end) {
        const suggested = suggestedBreakMinutes(start, end, date, null);
        const breakEl = $('ts-break');
        if (breakEl && Number(breakEl.value) !== suggested) {
            breakEl.value = String(suggested);
        }
    }
    const brk = Number($('ts-break')?.value || 0);
    const user = find(state.users, uid, 'uid');
    const wage = user ? Number(user.wage) || DEFAULT_WAGE : DEFAULT_WAGE;
    const work = calculateHours(start, end, date, null, brk);
    const total = work + bonus;
```

- [ ] **Step 5: Flag-Reset im `saveTimesheet`-Success-Pfad**

Suche im `saveTimesheet`-Success-Branch die Reset-Zeile mit `breakSelectHTML('ts-break', 0)`. Füge direkt nach dieser Zeile ein:

```js
            tsBreakManuallyTouched = false;
```

- [ ] **Step 6: Syntax-Check**

```bash
node /tmp/claude/syncheck.cjs
```

Expected: `OK`

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: auto-suggest legal break in timesheet modal"
```

---

## Task 3: Warn-Icon in Event-Detail-Timesheet-Liste

**Files:**
- Modify: `index.html` Event-Detail-Render (Zeilen 2735–2756)

- [ ] **Step 1: Pause-Zelle mit Warn-Icon erweitern**

Suche Zeile 2746:

```js
                <td>${Number(t.breakMinutes||0)} min</td>
```

Ersetze durch:

```js
                <td>${Number(t.breakMinutes||0)} min${missingLegalBreak(t) ? ` <span title="Gesetzliche Mindestpause: ${suggestedBreakMinutes(t.start,t.end,t.date,null)} min" style="color:var(--amber); font-weight:700;">⚠️</span>` : ''}</td>
```

- [ ] **Step 2: Syntax-Check**

```bash
node /tmp/claude/syncheck.cjs
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: warn icon in event timesheet list when legal break is missing"
```

---

## Task 4: Warn-Icon in User-Detail-Timesheet-Liste (`refreshUserDetail`)

**Files:**
- Modify: `index.html` `refreshUserDetail` (Zeilen 3952–3966)

- [ ] **Step 1: Pause-Zelle mit Warn-Icon erweitern**

Suche Zeile 3962:

```js
                <td>${Number(t.breakMinutes||0)} min</td>
```

Ersetze durch:

```js
                <td>${Number(t.breakMinutes||0)} min${missingLegalBreak(t) ? ` <span title="Gesetzliche Mindestpause: ${suggestedBreakMinutes(t.start,t.end,t.date,null)} min" style="color:var(--amber); font-weight:700;">⚠️</span>` : ''}</td>
```

- [ ] **Step 2: Syntax-Check**

```bash
node /tmp/claude/syncheck.cjs
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: warn icon in user timesheet detail when legal break is missing"
```

---

## Task 5: `pdfSignatureBlock` — Links-Layout für `withOrt`

**Files:**
- Modify: `index.html` `pdfSignatureBlock` (Zeilen 1511–1542)

- [ ] **Step 1: Komplette Funktion ersetzen**

Aktueller Stand Zeilen 1511–1542:

```js
function pdfSignatureBlock(doc, startY, opts = {}) {
    const pageH = doc.internal.pageSize.getHeight();
    const needed = opts.withDisclaimer ? 55 : 30;
    let y = startY;
    if (y + needed > pageH - 25) {
        doc.addPage();
        y = 30;
    }
    if (opts.withDisclaimer) {
        doc.setFont('helvetica', 'normal');
        doc.setFontSize(10);
        doc.setTextColor(0);
        const txt = 'Die Aushilfe bestätigt hiermit, dass sie auf eigenen Namen und Rechnung handelt und noch andere Arbeitgeber hat.';
        const lines = doc.splitTextToSize(txt, 180);
        doc.text(lines, 15, y);
        y += lines.length * 5 + 12;
    }
    doc.setFontSize(9);
    doc.setTextColor(0);
    const leftLabel = opts.leftLabel || 'Mitarbeiter';
    doc.text('_______________________', 15, y);
    doc.text(leftLabel, 15, y + 5);
    if (opts.withOrt) {
        doc.text('_______________________', 80, y);
        doc.text('Ort', 80, y + 5);
        doc.text('_______________________', 145, y);
        doc.text('Datum', 145, y + 5);
    } else {
        doc.text('_______________________', 110, y);
        doc.text('Datum', 110, y + 5);
    }
}
```

Ersetze durch:

```js
function pdfSignatureBlock(doc, startY, opts = {}) {
    const pageH = doc.internal.pageSize.getHeight();
    const needed = opts.withDisclaimer ? (opts.withOrt ? 75 : 55) : (opts.withOrt ? 50 : 30);
    let y = startY;
    if (y + needed > pageH - 25) {
        doc.addPage();
        y = 30;
    }
    if (opts.withDisclaimer) {
        doc.setFont('helvetica', 'normal');
        doc.setFontSize(10);
        doc.setTextColor(0);
        const txt = 'Die Aushilfe bestätigt hiermit, dass sie auf eigenen Namen und Rechnung handelt und noch andere Arbeitgeber hat.';
        const lines = doc.splitTextToSize(txt, 180);
        doc.text(lines, 15, y);
        y += lines.length * 5 + 12;
    }
    doc.setFontSize(9);
    doc.setTextColor(0);
    const leftLabel = opts.leftLabel || 'Mitarbeiter';
    if (opts.withOrt) {
        doc.text('_______________', 15, y);
        doc.text('Ort', 15, y + 5);
        doc.text('_______________', 55, y);
        doc.text('Datum', 55, y + 5);
        y += 18;
        doc.text('_______________________', 15, y);
        doc.text(leftLabel, 15, y + 5);
    } else {
        doc.text('_______________________', 15, y);
        doc.text(leftLabel, 15, y + 5);
        doc.text('_______________________', 110, y);
        doc.text('Datum', 110, y + 5);
    }
}
```

- [ ] **Step 2: Syntax-Check**

```bash
node /tmp/claude/syncheck.cjs
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: left-align pdfSignatureBlock withOrt layout"
```

---

## Task 6: `exportEvent` — Block-Layout pro Mitarbeiter

**Files:**
- Modify: `index.html` `exportEvent` (Zeilen 1634–1704)

- [ ] **Step 1: Komplette Funktion ersetzen**

Aktueller Stand Zeilen 1634–1704 (gesamte `exportEvent`-Funktion bis zum schließenden `}` vor `function exportRange`):

```js
function exportEvent(eventId) {
    const ev = find(state.events, eventId);
    if (!ev) return showToast('Event nicht gefunden', 'error');
    const list = timesheetsByEvent(eventId);
    const doc = pdfDoc();
    if (!doc) return;
    const subtitle = ev.endDate && ev.endDate !== ev.date
        ? `${fmtDateShort(ev.date)} – ${fmtDateShort(ev.endDate)}`
        : fmtDateShort(ev.date || '');
    const startY = pdfHeader(doc, `Event: ${ev.title || ''}`, subtitle);
    doc.setFontSize(9);
    doc.text(`Typ: ${ev.type || '-'}`, 15, startY);
    doc.text(`Ort: ${ev.ort || '-'}`, 15, startY + 5);
    doc.text(`Personen: ${ev.personen || '-'}`, 15, startY + 10);

    const byUser = {};
    list.forEach(t => {
        if (!byUser[t.userId]) byUser[t.userId] = [];
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
                fmtDateShort(t.date),
                `${t.start || '-'}–${t.end || '-'}`,
                `${Number(t.breakMinutes || 0)} min`,
                pdfHours(t.workHours),
                pdfHours(t.totalHours),
                pdfCurrency(t.totalCost)
            ];
            row._over = over;
            body.push(row);
        });
        body.push([
            { content: `Summe ${u.name}`, colSpan: 5, styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
            { content: pdfHours(subH), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
            { content: pdfCurrency(subC), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } }
        ]);
    });
    doc.autoTable({
        startY: startY + 18,
        margin: { bottom: 25 },
        head: [['Mitarbeiter','Datum','Zeit','Pause','Arbeit','Gesamt','Kosten']],
        body,
        theme: 'grid',
        headStyles: PDF_HEAD_STYLES,
        styles: PDF_STYLES,
        didParseCell(data) {
            if (data.section === 'body' && data.row.raw && data.row.raw._over) {
                data.cell.styles.fillColor = [220, 220, 220];
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

Ersetze durch:

```js
function exportEvent(eventId) {
    const ev = find(state.events, eventId);
    if (!ev) return showToast('Event nicht gefunden', 'error');
    const list = timesheetsByEvent(eventId);
    const doc = pdfDoc();
    if (!doc) return;
    const subtitle = ev.endDate && ev.endDate !== ev.date
        ? `${fmtDateShort(ev.date)} – ${fmtDateShort(ev.endDate)}`
        : fmtDateShort(ev.date || '');
    const startY = pdfHeader(doc, `Event: ${ev.title || ''}`, subtitle);
    doc.setFontSize(9);
    doc.text(`Typ: ${ev.type || '-'}`, 15, startY);
    doc.text(`Ort: ${ev.ort || '-'}`, 15, startY + 5);
    doc.text(`Personen: ${ev.personen || '-'}`, 15, startY + 10);

    if (!list.length) {
        doc.setFontSize(11);
        doc.text('Keine Schichten erfasst.', 15, startY + 22);
        pdfFooter(doc);
        doc.save(pdfFilename('event', [ev.title || eventId, ev.date || '']));
        return;
    }

    const byUser = {};
    list.forEach(t => {
        if (!byUser[t.userId]) byUser[t.userId] = [];
        byUser[t.userId].push(t);
    });
    const entries = Object.entries(byUser).map(([uid, arr]) => {
        const u = find(state.users, uid, 'uid') || { name: 'Gelöschter Mitarbeiter', wage: DEFAULT_WAGE };
        arr.sort((a, b) => (a.date || '').localeCompare(b.date || ''));
        return { uid, user: u, shifts: arr };
    });
    entries.sort((a, b) => (a.user.name || '').localeCompare(b.user.name || '', 'de'));

    let totalH = 0, totalC = 0;
    let currentY = startY + 18;
    const pageH = doc.internal.pageSize.getHeight();

    entries.forEach(({ user: u, shifts }) => {
        if (currentY + 40 > pageH - 25) {
            doc.addPage();
            currentY = 30;
        }
        doc.setFont('helvetica', 'bold');
        doc.setFontSize(11);
        doc.setTextColor(0);
        doc.text(u.name || 'Gelöschter Mitarbeiter', 15, currentY);
        doc.setFont('helvetica', 'normal');
        doc.setFontSize(9);
        doc.setTextColor(100);
        doc.text(`Stundenlohn: ${pdfCurrency(u.wage || DEFAULT_WAGE)}`, 15, currentY + 5);
        doc.setTextColor(0);

        let subH = 0, subC = 0;
        const body = shifts.map(t => {
            subH += Number(t.totalHours) || 0;
            subC += Number(t.totalCost) || 0;
            totalH += Number(t.totalHours) || 0;
            totalC += Number(t.totalCost) || 0;
            const over = getDailyHours(t.userId, t.date) > 10;
            const row = [
                fmtDateShort(t.date),
                `${t.start || '-'}–${t.end || '-'}`,
                `${Number(t.breakMinutes || 0)} min`,
                pdfHours(t.workHours),
                pdfHours(t.totalHours),
                pdfCurrency(t.totalCost)
            ];
            row._over = over;
            return row;
        });
        doc.autoTable({
            startY: currentY + 9,
            margin: { bottom: 25 },
            head: [['Datum','Zeit','Pause','Arbeit','Gesamt','Kosten']],
            body,
            theme: 'grid',
            headStyles: PDF_HEAD_STYLES,
            styles: PDF_STYLES,
            columnStyles: {
                0: { halign: 'center' },
                3: { halign: 'right' },
                4: { halign: 'right' },
                5: { halign: 'right' }
            },
            didParseCell(data) {
                if (data.section === 'body' && data.row.raw && data.row.raw._over) {
                    data.cell.styles.fillColor = [220, 220, 220];
                }
            },
            foot: [[
                { content: `Summe ${u.name}`, colSpan: 4, styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
                { content: pdfHours(subH), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } },
                { content: pdfCurrency(subC), styles: { halign: 'right', fontStyle: 'bold', fillColor: [240,240,240] } }
            ]]
        });
        currentY = doc.lastAutoTable.finalY + 8;
    });

    if (currentY + 20 > pageH - 25) {
        doc.addPage();
        currentY = 30;
    }
    doc.autoTable({
        startY: currentY + 4,
        margin: { bottom: 25 },
        body: [[
            { content: 'Event-Summe', colSpan: 4, styles: { halign: 'right', fontStyle: 'bold', fillColor: [0,0,0], textColor: 255 } },
            { content: pdfHours(totalH), styles: { halign: 'right', fontStyle: 'bold', fillColor: [0,0,0], textColor: 255 } },
            { content: pdfCurrency(totalC), styles: { halign: 'right', fontStyle: 'bold', fillColor: [0,0,0], textColor: 255 } }
        ]],
        theme: 'grid',
        styles: PDF_STYLES
    });

    pdfSignatureBlock(doc, doc.lastAutoTable.finalY + 15, { withDisclaimer: true, leftLabel: 'Aushilfe', withOrt: true });
    pdfFooter(doc);
    doc.save(pdfFilename('event', [ev.title || eventId, ev.date || '']));
}
```

- [ ] **Step 2: Syntax-Check**

```bash
node /tmp/claude/syncheck.cjs
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: rebuild exportEvent as per-employee block layout"
```

---

## Task 7: Manuelle Testrunde

Führe alle Tests aus dem Testplan der Spec durch. Lokaler Static-Server z. B. `python3 -m http.server 8080`, Login mit Admin-Rolle.

- [ ] **Step 1: Quittung-PDF Layout**

Schicht im Event-Detail bearbeiten, unten auf „Quittung" klicken. Im erzeugten PDF prüfen:
- Disclaimer-Text komplett lesbar, linksbündig
- Unterhalb: Ort-Bar bei x=15 mm, Datum-Bar bei x=55 mm, nebeneinander
- Darunter (ca. 18 mm Abstand): Aushilfe-Bar bei x=15 mm, volle Länge
- Alles auf linker Seitenhälfte, keine Elemente rechts von ~80 mm

- [ ] **Step 2: Monats-PDF unverändert**

„Personal & Zeiten" → User → Monat → PDF. Signatur-Zeile muss wie vorher aussehen (Mitarbeiter bei x=15, Datum bei x=110).

- [ ] **Step 3: Pause-Auto-Vorschlag 6–9 h**

Neuen Timesheet-Eintrag anlegen: Start 16:00, Ende 23:00. Pause-Select springt automatisch auf 30 min. Preview zeigt 6.50 h Arbeit.

- [ ] **Step 4: Pause-Auto-Vorschlag > 9 h**

Start 14:00, Ende 00:00 (Mitternacht-Übergang). Pause-Select springt auf 45 min. Preview zeigt 9.25 h Arbeit.

- [ ] **Step 5: Pause-Auto-Vorschlag ≤ 6 h**

Start 18:00, Ende 22:00. Pause bleibt auf 0 min. Preview zeigt 4.00 h.

- [ ] **Step 6: Manueller Override wird respektiert**

Start 18:00, Ende 22:00. Pause manuell auf 45 min stellen. Dann Start auf 14:00 ändern. Pause bleibt 45 (wird NICHT auf 0 zurückgesetzt und NICHT auf 30 hochgesetzt).

- [ ] **Step 7: Edit-Modus überschreibt nicht**

Bestehenden Eintrag mit 0 min Pause und 8 h Arbeit öffnen (Bearbeiten-Button). Pause zeigt weiter 0 min (wird NICHT auf 30 gesetzt).

- [ ] **Step 8: Retro-Warn-Icon Event-Detail**

Im Event-Detail-Modal einen Eintrag mit 8 h Arbeit + 0 min Pause finden. Pause-Zelle zeigt `0 min ⚠️`. Mouse-Hover zeigt Tooltip „Gesetzliche Mindestpause: 30 min".

- [ ] **Step 9: Retro-Warn-Icon User-Detail**

„Personal & Zeiten" → User-Detail-Modal → Event mit dem gleichen Eintrag → Pause-Spalte zeigt Warn-Icon.

- [ ] **Step 10: Event-PDF einfaches Event**

Event mit 3 verschiedenen MAs, je eine Schicht → PDF-Export:
- 3 Blöcke übereinander, alphabetisch sortiert
- Jeder Block hat Name (bold) + „Stundenlohn: X,XX €" (grau)
- Tabelle mit 6 Spalten (Datum/Zeit/Pause/Arbeit/Gesamt/Kosten)
- MA-Zwischensumme als graue foot-Zeile
- Event-Summe in schwarzem Block am Ende
- Signatur-Block unten mit Ort/Datum/Aushilfe im Links-Layout

- [ ] **Step 11: Event-PDF mehrtägig, 1 MA 2 Schichten**

MA A arbeitet an Tag 1 und Tag 2 desselben Events. PDF-Export:
- 1 Block für MA A
- Tabelle zeigt beide Schichten untereinander, nach Datum sortiert
- MA-Summe summiert beide Tage korrekt

- [ ] **Step 12: Event-PDF 0 Schichten**

Event ohne Zeiterfassungen → PDF-Export → Header + „Keine Schichten erfasst." + Footer. Kein Crash.

- [ ] **Step 13: Event-PDF viele MAs (Seitenumbruch)**

Event mit 6+ MAs → PDF. Seitenumbruch zwischen Blöcken sauber, kein Block mit Header auf einer Seite und Tabelle auf der nächsten.

- [ ] **Step 14: Fix + Re-Test bei Findings**

Wenn alle Tests passen, keine weiteren Commits nötig. Bei Findings: Fix + Commit, dann Testrunde im betroffenen Bereich wiederholen.

---

## Task 8: Push zum Remote

- [ ] **Step 1: Git Status prüfen**

```bash
git status
```

Expected: `nothing to commit, working tree clean`.

- [ ] **Step 2: Push**

```bash
git push origin main
```

Expected: Commits landen auf GitHub.

---

## Self-Review Notes

- **Spec-Abdeckung:** Komponente A → Task 5. Komponente B → Tasks 1, 2, 3, 4. Komponente C → Task 6. Manueller Testplan → Task 7. Alle Spec-Requirements gedeckt.
- **Placeholder-Scan:** Keine TBD/TODO, alle Code-Blöcke komplett.
- **Konsistenz:** `suggestedBreakMinutes`, `missingLegalBreak`, `tsBreakManuallyTouched` — Namen konsistent über alle Tasks. `pdfSignatureBlock`-Signatur bleibt kompatibel zu allen bestehenden Aufrufern (Monats- und Range-PDF nutzen den `else`-Zweig unverändert).
- **Edge-Case „Pause = 0 manuell trotz Regel":** Warn-Icon erscheint trotzdem in Listen (Retro-Sicht) — gewollt als Dokumentations-Signal.
- **Kein neues DOM-Assignment-Pattern:** Task 3 und Task 4 modifizieren bestehende Template-Literals innerhalb bereits existierender Zuweisungen — keine neuen.

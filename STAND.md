# Form-Check — Aktueller Stand & Projekt-Infos

> Stand: 2026-06-27 · Eigenständige Pose-Detection-Trainings-App, läuft komplett im Browser.
> Deutsch, Du-Anrede, Standalone-HTML, kein Build-Step, Deployment über GitHub Pages.

---

## ⚠️ Wichtigste Arbeitsregel: IMMER auf `main` pushen

**Alle Änderungen werden ausschließlich auf den `main`-Branch committet und gepusht.**
Keine Feature-Branches, keine anderen Branches. Wenn ich (Claude) an der App arbeite,
gilt: entwickeln → committen → `git push origin main`. Punkt.

```bash
git add index.html
git commit -m "kurze, klare Beschreibung"
git push origin main
```

- Repo: `nicolehahn2890/Formcheck`
- Deployment: GitHub Pages, Branch `main`, Root → URL **https://nicolehahn2890.github.io/Formcheck/**
- Push auf `main` = sofort live (paar Minuten Cache).

---

## 1. Was die App ist

Eine **Pose-Detection-Trainings-App** im Browser: misst über die Kamera Gelenkwinkel,
bewertet die Bewegungstiefe und **zählt Wiederholungen automatisch**. Kein Server,
kein API-Key, alles lokal.

- **Zielgerät:** iPhone (Safari), zusätzlich Desktop zum Testen.
- **Kamera braucht HTTPS** → läuft nur über GitHub Pages (`https://…`) oder `localhost`,
  **nicht** per Doppelklick auf die Datei am iPhone.
- **Eigenständig — KEINE Peach-Anbindung.** Das Profil liegt in einem eigenen
  localStorage-Key `formcheck_profile`, getrennt von Peachs `peach_v4`. Eine spätere
  Anbindung ist bewusst NICHT eingebaut.

## 2. Tech-Stack (genau diese CDN-Versionen, erprobt)

```html
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-core@4.22.0/dist/tf-core.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-converter@4.22.0/dist/tf-converter.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-backend-webgl@4.22.0/dist/tf-backend-webgl.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/pose-detection@2.1.3/dist/pose-detection.min.js"></script>
```

- Modell: **MoveNet**, `SINGLEPOSE_LIGHTNING`, Backend `webgl`.
- 17 COCO-Keypoints, jeder mit `x`, `y`, `score`. Nur Punkte mit `score > 0.3` verwenden.
- Fonts: **Archivo Black** (Display) + **Space Grotesk** (Body/UI) via Google Fonts.
- Alles in einer Datei: **`index.html`** (HTML + CSS + JS inline). Kein Build, kein npm.

## 3. Architektur (der Kern-Trick)

**Jede Übung ist nur ein Config-Objekt. Die Logik ist für alle Übungen identisch.**

Jeder Winkel wird in einen **Tiefen-Prozentwert (0..1)** umgerechnet. In diesem pct-Raum
gilt für JEDE Übung „mehr = besser/tiefer" → der Zustandsautomat ist universell.
Neue Übung = neue Zeile in `EXERCISES`, kein neuer Code.

```
Winkel → pct (0..1):
  type 'low'  (Arbeitsphase = KLEINER Winkel): pct = (stand - ang) / (stand - deep)
  type 'high' (Arbeitsphase = GROSSER Winkel): pct = (ang - stand) / (top - stand)
  → immer auf 0..1 clampen
```

**Universeller Zustandsautomat** (im pct-Raum, pro Frame mit aktuellem `pct`):
- `workExtremePct = max(workExtremePct, pct)` (Extrem des aktuellen Reps)
- `pct > enterPct` und `stage==='ready'` → `stage='working'`
- `pct < exitPct` und `stage==='working'` → **Rep fertig**: `reps++`, Qualität aus
  `workExtremePct` bestimmen, in `repLog` schreiben, Piep, zurücksetzen.
- Qualität: `>= goodPct` → gut · `>= okPct` → fast · sonst → zu wenig.

Wichtige Funktionen in `index.html`:
- `angleAt(a,b,c)` — Winkel am mittleren Punkt b (Grad)
- `angleToPct(ex, ang)` — Winkel → Tiefen-Prozent je Typ
- `pickJoints(keypoints, sides)` — config-getrieben, prüft beide Seiten, nimmt die mit
  besseren `score`-Werten (alle drei Punkte > 0.3), sonst `null`
- `setExercise(key)` — aktive Config setzen, pct-Schwellen vorberechnen (inkl. Level-Faktor),
  Session reset, UI aktualisieren
- `updateReps(pct, angle)` — Zustandsautomat (zählt nur während eines aktiven Satzes)

## 4. Übungen (aktuell 14)

Jede Übung hat eine Muskelgruppen-**Kategorie** (Farben aus dem Peach-Design-System) und
**Ziel-Tags** für die Empfehlung. Winkel in Grad, `sides` = ein Tripel pro Körperseite,
mittlerer Punkt = gemessenes Gelenk.

| key | Übung | Typ | Gelenk | Kategorie |
|-----|-------|-----|--------|-----------|
| squat | Kniebeuge | low | Knie | Glute & Quad |
| hipthrust | Hip Thrust | high | Hüfte | Glute Max |
| lunge | Ausfallschritt | low | Knie | Glute & Quad |
| rdl | Rumän. Kreuzheben | low | Hüfte | Glute & Hams |
| glutebridge | Glute Bridge | high | Hüfte | Glute Max |
| bulgarian | Bulgarian Split Squat | low | Knie | Glute & Quad |
| stepup | Step-up | low | Knie | Glute & Quad |
| goodmorning | Good Morning | low | Hüfte | Glute & Hams |
| pushup | Liegestütze | low | Ellbogen | Brust |
| shoulderpress | Schulterdrücken | high | Ellbogen | Schultern |
| dips | Trizeps-Dips | low | Ellbogen | Trizeps |
| row | Vorgeb. Rudern | low | Ellbogen | Rücken |
| curl | Bizeps-Curl | low | Ellbogen | Bizeps |
| crunch | Crunch (experimentell) | low | Rumpf | Bauch |

**Die Winkel-Werte sind Startpunkte** — die der 5 Ursprungsübungen aus dem Briefing,
die der 9 neuen biomechanisch plausibel geschätzt. **Nach echtem Test justieren** (Nicoles
Lernteil). Stellschrauben pro Übung in `EXERCISES`: `stand`, `deep`/`top`, `enter`, `exit`,
`good`, `ok`.

**Nicht jede Peach-Übung ist kamera-messbar:** Formcheck misst EINEN Gelenkwinkel aus der
Seitenansicht. Maschinen-Abduktion, Kabel-Kickbacks o.ä. lassen sich so nicht zuverlässig
abbilden — deshalb nur die gut messbaren ergänzt.

## 5. Profil (eigenständig, in `formcheck_profile`)

Aufklappbarer Block „Dein Profil" auf dem Startscreen. Wird live in localStorage gespeichert.

- **Körpergröße (cm):** wird gespeichert & angezeigt. Bewusst **NICHT** in die Winkel-Mathe
  eingerechnet — Gelenkwinkel sind anatomisch ~körpergrößenunabhängig (ein paralleler Squat
  ist bei jedem ~90 Grad Kniewinkel). Keine Pseudo-Genauigkeit.
- **Level (Anfänger / Mittel / Profi):** skaliert die **gut/fast-Schwellen** (= wie streng
  „tief genug" gilt). Faktor: Anfänger 0.88 · Mittel 1.0 · Profi 1.07 (auf `goodPct`/`okPct`,
  gedeckelt bei 0.97). Die **Zähl-Schwellen `enter`/`exit` bleiben level-unabhängig** — eine
  Wiederholung ist eine Wiederholung, nur die Qualitätsbewertung verschiebt sich.
- **Ziele (Glutefokus, Beinkraft, Oberkörper, Schmaler Oberkörper, Core/Bauch, Ganzkörper):**
  heben passende Übungen mit ★ hervor und sortieren sie nach vorne. Ändern die Mathe nicht.

## 6. Satz-Flow + Aufnahme/Replay

**Zweigleisig: Daten-Auswertung läuft IMMER, Video ist optional (Feature-Detection).**

- Großer Button unten: **„Satz starten" ↔ „Satz beenden"**. Reps werden nur während eines
  aktiven Satzes gezählt.
- **Video (optional):** `canvas.captureStream(30)` + `MediaRecorder`, Mime per `pickMime()`
  (iOS-MP4 vor WebM). Fehlt Unterstützung → reiner Daten-Modus + ehrlicher Hinweis.
- **Summary-Overlay** nach jedem Satz: Reps gesamt · „x von n tief genug" · Ø Tiefe · pro Rep
  ein nach Qualität eingefärbter Balken · bei Erfolg Video + Download-Link · „Neuer Satz".
- Übungswechsel mitten im Satz bricht die Aufnahme sauber ab.

**Kamera vorne/hinten:** Auf dem Startscreen wählbar (Hinten / Vorne-Selfie), im Lauf per
„Kamera"-Button umschaltbar (auch während eines Satzes). Die **Frontkamera wird gespiegelt**
dargestellt (natürliche Selfie-Ansicht); die Winkelzahl bleibt lesbar, die Winkel-Mathe ist
spiegel-invariant (Bewertung unverändert). Standard ist die Rückkamera.

## 7. Design-System (Neobrutalism, aus dem Peach-Projekt)

Übernommen als visuelle Sprache, übersetzt auf Formchecks Vollbild-Kamera-Oberflächen:

- **Tokens** auf `:root`: helle Farbblöcke, Ink (`#181016`) für alle Ränder/Schatten/Text auf
  Blöcken, Lavendel-Hintergrund (`#E9DEF8`).
- **Ränder** 2.5px solid Ink · **Schatten** hart ohne Blur (`4px 4px 0` etc.) · **Pills**
  (radius 999px) für Buttons/Bars · **Press-Effekt** (`translate(2px,2px)`, Schatten kollabiert).
- **Fonts:** Archivo Black (Titel, große Zahlen, UPPERCASE) + Space Grotesk (UI).
- **Kategorie-Farben** als Punkt pro Übungs-Chip.
- **Peach-Produktregel befolgt:** keine Grad-Zeichen (`°`) in JS-Strings — „Grad" ausgeschrieben.
  Alle Inputs `font-size:16px` (verhindert iOS-Auto-Zoom).

## 8. Datenstruktur (nur im Speicher, für evtl. späteren Export)

```js
session = {
  exercise: 'squat',
  date: '2026-06-27T…Z',
  reps: [ { extremePct: 0.84, quality: 'good', angle: 86 }, … ],
  totalReps: 8,
  goodReps: 6
}
```
Wird pro Satz im Speicher gehalten und als JSON in die Konsole geloggt. **Kein Export,
keine Peach-Anbindung.**

## 9. Bekannte Stolpersteine

- **HTTPS-Pflicht** für die Kamera → am iPhone nur über GitHub Pages, am PC `localhost`.
- iOS Safari: `<video>` braucht `playsinline` + `muted`; Kamerastart muss aus einer
  Nutzer-Geste (Button-Klick) kommen.
- **MediaRecorder auf iOS unzuverlässig** → deshalb die zweigleisige Lösung (Daten immer).
- MoveNet lädt beim ersten Start das Modell aus dem Netz (paar Sekunden) — Lade-Status wird angezeigt.
- Performance: Aufnahme + Pose-Detection parallel ist auf modernen iPhones ok, auf alten ggf. ruckelig.
- Foot/Ankle-Keypoints von MoveNet sind schwächer → Waden-/Knöchel-Übungen bewusst nicht eingebaut.

## 10. Testen & Entwickeln

- **Lokaler Logik-/Syntax-Test ohne Kamera:** das inline-`<script>` lässt sich mit gestubbtem
  DOM in Node ausführen (Zustandsautomat, Schwellen, Satz-Flow, Summary durchsimulieren).
- **Visueller Test:** Startscreen rendert ohne Kamera — per Headless-Browser screenshotbar.
- **Echter Test:** nur am Gerät über die Pages-URL (Kamera + HTTPS). Handy **seitlich**
  aufstellen, ganzer Körper / die gemessenen Gelenke im Bild.

## 11. Mögliche nächste Schritte

- Winkel-Startwerte der neuen Übungen nach echtem Test justieren.
- Core/Bauch ausbauen (Crunch ist experimentell; MoveNet hier weniger zuverlässig).
- Weitere kamera-messbare Übungen ergänzen (falls gewünscht — vorher Messbarkeit prüfen).
- Mehrere Sätze pro Session sammeln / Verlauf anzeigen (weiterhin lokal, ohne Peach).

---

## Dateien im Repo

- **`index.html`** — die komplette App (HTML + CSS + JS inline).
- **`BRIEFING-formcheck.md`** — ursprüngliches Handoff-Briefing (Architektur-Vorgabe).
- **`STAND.md`** — dieses Dokument (aktueller Stand).

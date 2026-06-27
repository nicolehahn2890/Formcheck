# Briefing: Form-Check App (Pose-Detection) — Weiterbau in Claude Code

> Handoff aus einem Claude-Chat. Du (Claude Code) baust an einer bestehenden MVP weiter.
> Nicoles Konventionen gelten: Deutsch, Du-Anrede, vollständiger sofort lauffähiger Code,
> keine Platzhalter. Standalone HTML/JS, Deployment über GitHub Pages (`nicolehahn2890`).
> Bei größeren Entscheidungen kurz Annahmen offenlegen, bevor du baust.

---

## 1. Was die App ist

Eine **Pose-Detection-Trainings-App**, die im Browser läuft (kein Server, kein API-Key, alles lokal).
Sie misst über die Kamera Gelenkwinkel, bewertet die Bewegungstiefe und **zählt Wiederholungen automatisch**.

- **Zielgerät:** iPhone (Safari), zusätzlich Desktop zum Testen.
- **Kamera braucht HTTPS** → läuft nur über GitHub Pages (`https://...`) oder `localhost`, nicht per Doppelklick auf dem iPhone.
- **Spätere Anbindung an „Peach"** (Nicoles Fitness-Tracker, `nicolehahn2890.github.io/Trainingsapp`, localStorage-Key `peach_v4`) ist geplant — Daten dafür sauber vorhalten, aber NOCH NICHT integrieren.

## 2. Aktueller Stand (MVP, liegt als `index.html` vor)

Funktioniert bereits für **eine** Übung (Kniebeuge):
- Holt Kamerastream (`getUserMedia`, Rückkamera).
- Lädt **MoveNet** (TensorFlow.js), schätzt pro Frame 17 Körperpunkte.
- Berechnet den **Kniewinkel** (Hüfte–Knie–Knöchel), wählt automatisch die besser sichtbare Seite.
- Zustandsautomat zählt Reps; **Piep pro Rep**; Tiefen-Balken + Live-Winkel; Tiefen-Feedback pro Rep.
- Zeichnet Videobild + Skelett Frame für Frame aufs Canvas.

## 3. Tech-Stack (genau diese CDN-Versionen, sind erprobt)

```html
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-core@4.22.0/dist/tf-core.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-converter@4.22.0/dist/tf-converter.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-backend-webgl@4.22.0/dist/tf-backend-webgl.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/pose-detection@2.1.3/dist/pose-detection.min.js"></script>
```

Modell: `MoveNet`, `modelType = SINGLEPOSE_LIGHTNING`. Backend `webgl`.
Keypoint-Namen (COCO/17): `left_hip`, `left_knee`, `left_ankle`, `right_*`, `left_shoulder`, `left_elbow`, `left_wrist`, `right_*` usw. Jeder Punkt hat `x`, `y`, `score`. Nur Punkte mit `score > 0.3` verwenden.

## 4. KERN-ARCHITEKTUR (bitte exakt so umsetzen)

**Prinzip: jede Übung ist nur ein Config-Objekt. Die Logik ist für alle Übungen identisch.**
Der Schlüssel: jeden Winkel in einen **Tiefen-Prozentwert (0..1)** umrechnen. In diesem pct-Raum
ist „mehr = besser/tiefer" für JEDE Übung gleich — dadurch wird der Zustandsautomat universell.

### 4a. Winkel-Mathe

```js
// Winkel am mittleren Punkt b, in Grad
function angleAt(a,b,c){
  const abx=a.x-b.x, aby=a.y-b.y, cbx=c.x-b.x, cby=c.y-b.y;
  const dot=abx*cbx+aby*cby;
  const mag=(Math.hypot(abx,aby)*Math.hypot(cbx,cby))||1;
  return Math.acos(Math.max(-1,Math.min(1,dot/mag)))*180/Math.PI;
}
```

### 4b. Winkel → Tiefen-Prozent (pro Übungs-Typ)

```js
// type 'low'  : Arbeitsphase = KLEINER Winkel (Squat, Pushup, Curl, Lunge)
//   pct = (standAngle - ang) / (standAngle - deepAngle)
// type 'high' : Arbeitsphase = GROSSER Winkel (Hip Thrust, Lockout)
//   pct = (ang - standAngle) / (topAngle - standAngle)
// Ergebnis immer auf 0..1 clampen.
```

### 4c. Universeller Zustandsautomat (im pct-Raum, gilt für ALLE Übungen)

- Pro Übung werden `enter`/`exit`/`good`/`ok` als **Winkel** definiert und einmalig per 4b in pct umgerechnet
  (`enterPct`, `exitPct`, `goodPct`, `okPct`).
- Ablauf je Frame mit aktuellem `pct`:
  - `workExtremePct = max(workExtremePct, pct)` (das tiefste/höchste des aktuellen Reps)
  - wenn `pct > enterPct` und `stage==='ready'` → `stage='working'`
  - wenn `pct < exitPct` und `stage==='working'` → **Rep fertig**: `reps++`, Qualität aus `workExtremePct`
    bestimmen, in `repLog` schreiben, Piep, `workExtremePct=0`, `stage='ready'`.
- Qualität: `workExtremePct >= goodPct` → gut · `>= okPct` → fast · sonst → zu wenig.

## 5. Die Übungs-Configs (konkrete Startwerte — als Tabelle, in Code überführen)

Alle Winkel in Grad. `sides` = je ein Tripel pro Körperseite; Winkel wird am **mittleren** Punkt gemessen.
Seitenwahl automatisch nach Mittelwert der `score` (alle drei Punkte > 0.3 verlangen).

| key       | label          | type | sides (mittlerer Punkt = Gelenk)                | stand | deep/top | enter | exit | good | ok  |
|-----------|----------------|------|-------------------------------------------------|-------|----------|-------|------|------|-----|
| squat     | Kniebeuge      | low  | hip–**knee**–ankle                              | 172   | 72       | 110   | 150  | 90   | 110 |
| hipthrust | Hip Thrust     | high | shoulder–**hip**–knee                           | 90    | 178      | 160   | 120  | 170  | 155 |
| lunge     | Ausfallschritt | low  | hip–**knee**–ankle                              | 172   | 80       | 115   | 150  | 95   | 115 |
| pushup    | Liegestütze    | low  | shoulder–**elbow**–wrist                        | 165   | 80       | 105   | 150  | 95   | 115 |
| curl      | Bizeps-Curl    | low  | shoulder–**elbow**–wrist                        | 160   | 40       | 80    | 140  | 50   | 70  |

`sides` Beispiel squat: `[['left_hip','left_knee','left_ankle'],['right_hip','right_knee','right_ankle']]`.
Feedback-Texte: type `low` → „tiefer", type `high` → „höher strecken" (für „fast"-Stufe).
Jede Config zusätzlich mit einem kurzen `cue` (Aufstell-Hinweis), z.B. squat: „Seitlich · ganzer Körper im Bild".

Diese Werte sind Startpunkte — Nicole justiert sie nach echtem Test (das ist explizit ihr Lernteil).

## 6. AUFGABE 1 — Umbau auf mehrere Übungen

- `EXERCISES`-Objekt nach Abschnitt 5 anlegen.
- `setExercise(key)`: aktive Config setzen, `enterPct/exitPct/goodPct/okPct` per 4b vorberechnen,
  Session zurücksetzen (reps, stage, workExtremePct, repLog), UI aktualisieren (cue, Übungs-Name,
  Tiefen-Marke im Balken auf `goodPct` setzen).
- `pickJoints(keypoints, sides)` config-getrieben (beide Seiten prüfen, bessere wählen, sonst `null`).
- **UI:** Übungs-Auswahl auf dem Startscreen (Chips oder `<select>`) und während des Laufens
  (kompaktes `<select>` in der oberen Leiste). Wechsel resettet die Session.

## 7. AUFGABE 2 — Aufnahme + Replay (mit iOS-Absicherung)

**Zweigleisig bauen — die Daten-Auswertung MUSS immer funktionieren, das Video ist optional.**

### 7a. Satz-Flow
- Großer Button unten: „Satz starten" ↔ „Satz beenden".
- Reps werden nur gezählt, solange ein Satz aktiv ist.
- Bei „Satz starten": Session reset + (falls unterstützt) Aufnahme starten.
- Bei „Satz beenden": Aufnahme stoppen → **Summary-Overlay** zeigen.

### 7b. Video-Aufnahme (optional, Feature-Detection!)
- Quelle: `canvas.captureStream(30)` (das Canvas enthält bereits Video + Skelett).
- `MediaRecorder` mit Mime-Detection — iOS Safari kann oft nur MP4, kein WebM:
  ```js
  function pickMime(){
    const cands=['video/mp4;codecs=h264','video/mp4','video/webm;codecs=vp9','video/webm;codecs=vp8','video/webm'];
    if(!('MediaRecorder' in window)) return '';
    return cands.find(m=>MediaRecorder.isTypeSupported(m)) || '';
  }
  ```
- Wenn kein Mime / kein `MediaRecorder` / `captureStream` fehlt → `recordingSupported=false`,
  Video überspringen, im Summary ehrlich hinweisen: „Video-Aufnahme auf diesem Gerät nicht unterstützt — hier deine Daten."
- Bei Erfolg: `Blob` aus Chunks, `URL.createObjectURL`, im Summary ein `<video controls playsinline>`
  plus Download-Link (`<a download>` mit passender Endung aus dem Mime).

### 7c. Daten-Auswertung (immer, geräteunabhängig)
Aus `repLog` (pro Rep: `extremePct`, Qualitätsstufe) im Summary zeigen:
- Anzahl Reps gesamt.
- Pro Rep ein kleiner Balken, eingefärbt nach Qualität (gut/fast/zu wenig).
- Zähl-Tally: „6 von 8 tief genug".
- Durchschnittliche Tiefe (Mittelwert `extremePct`).
- Button „Neuer Satz".

## 8. Datenstruktur für spätere Peach-Anbindung (nur vorhalten)

Session-Objekt sammeln, damit später ein Export nach Peach (`peach_v4`) leicht möglich ist:
```js
const session = {
  exercise: 'squat',
  date: new Date().toISOString(),
  reps: [ { extremePct: 0.84, quality: 'good', kneeAngleMin: 86 }, /* ... */ ],
  totalReps: 8,
  goodReps: 6
};
```
Noch NICHT in Peach schreiben — nur im Speicher halten / optional als JSON loggen.

## 9. Deployment & Test

1. Datei `index.html` ins Repo (z.B. `FormCheck`, Public).
2. Settings → Pages → Branch `main`, `/root`, Save.
3. URL `https://nicolehahn2890.github.io/FormCheck/` auf dem **iPhone** öffnen, Kamera erlauben.
4. Handy **seitlich** aufstellen, ganzer Körper im Bild.

## 10. Bekannte Stolpersteine

- **HTTPS-Pflicht** für Kamera (s.o.) — lokal am PC `localhost` ok, iPhone nur über Pages.
- iOS Safari: `<video>` braucht `playsinline` + `muted` fürs Autoplay; Kamerastart muss aus einer
  Nutzer-Geste (Button-Klick) kommen.
- **MediaRecorder auf iOS ist unzuverlässig** — deshalb die zweigleisige Lösung aus Abschnitt 7.
- MoveNet lädt beim ersten Start das Modell aus dem Netz (paar Sekunden) — Lade-Status anzeigen.
- Performance: Aufnahme + Pose-Detection parallel ist auf modernen iPhones ok, auf alten ggf. ruckelig.

## 11. Reihenfolge-Empfehlung

Erst **Aufgabe 1** (Config-Refactor, ändert die Architektur), dann **Aufgabe 2** (Aufnahme/Replay
oben drauf, nimmt automatisch die aktive Übung auf). Nach jedem Schritt lauffähig halten.

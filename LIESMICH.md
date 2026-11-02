# Zeitquader

Slit-Scan, Zeitversatz und der Video-Quader in einem Projekt, weil alle drei dasselbe sind: verschiedene Schnitte durch einen Block aus Raum und Zeit. Ein eigenes Repo wie die anderen. Zum Einbauen ins Portfolio gibt es `merge-zeitquader.zip`.

**Einrichten**
- Inhalt in ein neues Repo `zeitquader` legen.
- `css/fonts/` aus dem Portfolio dazukopieren.
- Mit deinem lokalen Server ausliefern.

**Bedienung in Kürze**
1. Video öffnen (oder „Kamera“ für Schlitz und Zeitversatz live).
2. Ansicht wählen, Bereich einstellen, „Rechnen“.
3. Beim Quader: ziehen dreht, Mausrad zoomt. Schnitt und Darstellung wirken sofort.
4. Export: „Video rendern“ macht beim Quader eine Kamerafahrt, beim Schlitz ein wachsendes Bild, beim Zeitversatz das Video Bild für Bild.

---

## 1. Ein Video ist ein Quader

Leg alle Bilder eines Videos hintereinander wie Scheiben Brot. Dann hast du einen Block: x und y sind das Bild, die dritte Richtung ist die Zeit t. Ein normales Video zeigt diesen Block Scheibe für Scheibe, also immer einen Schnitt bei t = jetzt.

Jeder **andere Schnitt** ist ein anderes Bild:

| Schnitt | Ergebnis | Ansicht |
|---|---|---|
| t = fest | ein normales Bild | (Quader, Schnitt „quer“) |
| x = fest | aus jedem Bild dieselbe Spalte, nebeneinander: **Slit-Scan / Zielfoto** | Schlitz |
| eine schräge oder gekrümmte Fläche | jedes Pixel aus einer anderen Zeit: **Zeitversatz** | Zeitversatz |
| gar kein Schnitt, der ganze Block von außen | **Video-Quader** | Quader |

Darum ist es ein Projekt und nicht drei: Die Schnittfläche im Quader *ist* ein Schlitz. Stellst du beim Quader „längs, senkrecht“ ein und schiebst „Schnitt bei“, siehst du auf der Schnittfläche genau das Schlitzbild.

---

## 2. Schlitz (`SCHLITZ_SHADER`, `baueSchlitz`)

Aus jedem Bild wird nur eine Spalte genommen (bei „Lage“ 0,5 die mittlere) und rechts an das Schlitzbild angehängt. Was am Schlitz vorbeizieht, wird abgebildet, was stillsteht, wird zu waagerechten Streifen. So funktioniert eine Zielfotokamera beim Pferderennen. Deswegen sind Pferde darauf manchmal gestaucht oder gedehnt: Breite bedeutet Zeit, und wer langsam ist, wird breit.

- **Dehnung:** Ein Video mit 300 Bildern gäbe nur 300 Pixel Breite. Darum ist jedes Bild in der Anzeige ein paar Pixel breit.
- **Waagerecht:** eine Zeile statt einer Spalte. Das Ergebnis läuft dann von oben nach unten.
- **Schreiben auf der GPU:** Der Shader zeichnet in einen `viewport`, der genau eine Spalte breit ist. Kein Pixel wandert über die CPU.

---

## 3. Zeitversatz (`VERSATZ_SHADER`)

Jedes Pixel holt sich seine Farbe aus einem anderen Bild im Ringpuffer. Wie weit zurück es greift, sagt ein **Muster** zwischen 0 und 1, mal „Tiefe“ in Bildern:

- *links jetzt, rechts früher:* Bewegungen ziehen sich schräg auseinander, Gesichter verbiegen sich beim Drehen.
- *Mitte jetzt, außen früher* (oder umgekehrt): wie ein Trichter in die Vergangenheit.
- *hell = früher:* das Bild selbst bestimmt die Zeit.
- *Welle:* wandert über das Bild.

Damit keine harten Stufen entstehen, mischt der Shader zwischen den zwei nächsten Bildern (`mix(a, b, e - floor(e))`). Live mit der Kamera ist das die schönste Ansicht: Man bewegt die Hand und sieht sie zerfließen.

Der Ringpuffer kommt aus `/lib/ringpuffer.js`, derselbe wie bei der Chronofotografie.

---

## 4. Quader (`QUADER_SHADER`, `kamera.js`)

### Geht das im Browser überhaupt? Und ergibt es Sinn?

Du kennst ihn aus Blender. Dort ist er meistens so gebaut: jedes Bild als **Fläche mit Textur**, hintereinander gestellt, halb durchsichtig. Das geht im Browser auch, hat aber zwei Probleme:
1. Durchsichtige Flächen müssen von hinten nach vorn gezeichnet werden. Dreht man die Kamera, stimmt die Reihenfolge nicht mehr.
2. Zwischen den Flächen ist nichts. Von der Seite sieht man Lamellen.

Darum mache ich es anders, mit **Raymarching**. Das ist dasselbe Verfahren, das du später für 3D-Fraktale brauchst:
- Für jedes Pixel der Anzeige wird ein Strahl von der Kamera aus geschickt (`zurueck` ist die umgekehrte Kameramatrix).
- Wo er den Block betritt und verlässt, rechnet die **Slab-Methode**: Ein Quader ist der Schnitt dreier Paare paralleler Ebenen. Für jedes Paar rechnet man, wann der Strahl hineinkommt und wann hinaus. Hinein ist er erst, wenn er in allen dreien drin ist (das Maximum der Eintritte), hinaus, sobald er eins verlässt (das Minimum der Austritte).
- Dazwischen 256 Schritte. An jedem wird aus dem Textur-Array gelesen, zwischen zwei Bildern gemischt. Es gibt also keine Lamellen, die Zeit ist durchgehend.

**Ergibt es Sinn?** Ehrlich: Als Bild ist der Quader schwerer lesbar als ein Schlitz. Man sieht vor allem die Ränder (die Außenseiten *sind* Schlitzbilder) und ein Leuchten im Innern. Stark wird er erst durch die Darstellung:

| Darstellung | was passiert | gut für |
|---|---|---|
| **nur Bewegtes** | Durchsichtig ist alles, was dem Median gleicht (wie beim Hintergrund-Abzug der Chronofotografie). Was sich bewegt, wird zu einer festen „Röhre“ durch die Zeit. | Personen, Autos, Vögel vor ruhigem Hintergrund. Das ist die eigentliche Idee. |
| **leuchten** | Alles wird nur aufaddiert, am Ende `1 - exp(-summe)` (damit es nicht ausbrennt) | Lichter in der Nacht, Feuerwerk |
| **Nebel** | alles ein wenig undurchsichtig, vorn verdeckt hinten | um den Block als Ding zu zeigen |

**Grenzen:** Alle Bilder müssen gleichzeitig auf der Grafikkarte liegen: 96 Bilder in 384 × 216 Pixeln sind etwa 30 MB. Darum gibt es den Quader nur mit Datei, nicht live, und nicht in voller Auflösung. Für ein großes Standbild ist Blender besser. Was der Browser bietet: Man dreht in Echtzeit, schneidet auf und probiert Schwellen aus.

### Schnitt

Eine Ebene durch den Block, die undurchsichtig ist: Der Strahl sammelt bis zu ihr, dort nimmt er das Bild der Schnittfläche und hört auf. Längs geschnitten (x oder y) ist die Fläche ein Schlitzbild, quer geschnitten (Zeit) ein einzelnes Bild. Mit „Schnitt bei“ fährst du durch.

### Kamera (`kamera.js`)

Eine Orbitkamera: Drehung und Neigung sind zwei Winkel auf einer Kugel um den Mittelpunkt, Abstand ist ihr Radius. Daraus entsteht eine Blickmatrix (lookAt) und eine Perspektive; der Shader bekommt deren Umkehrung. Du hattest geschrieben, dass du eine freie Kamera später für Fraktale, Punktwolken und Gaussian Splats willst. Dann kommt hier die Bewegung mit Tasten dazu (W/A/S/D in Blickrichtung), die Mathematik bleibt dieselbe. Die „Taschenlampe“ wäre ein Kegel um `richtung` aus Kameraposition: im Shader `smoothstep` auf `dot(normalize(p - auge), blick)`.

---

## 5. Dateien

| Datei | Aufgabe |
|---|---|
| `js/main.js` | Quelle (Datei/Kamera), die drei Abläufe, Ziehen und Zoomen, Export |
| `js/gpu.js` | WebGL2: Ringpuffer, Schlitzbild, Versatz, Median, Quader |
| `js/shader.js` | alle Shader, oben die Erklärung |
| `js/kamera.js` | Orbitkamera → `auge` und `zurueck` |
| `lib/ringpuffer.js`, `lib/video-quelle.js` | gemeinsam mit Chronofotografie und Datamosh |

---

## 6. Ehrlich

- **Kamera live** geht für Schlitz und Zeitversatz. Beim Quader müsste man alle Bilder neu hochladen, sobald eins dazukommt. Machbar wäre ein wandernder Quader (Ringpuffer, Ebene 0 springt), aber das habe ich gelassen.
- **Handys:** Der Ringpuffer ist auf dem Handy auf 300 MB begrenzt, der Zeitversatz hat dann weniger Tiefe. Der Quader mit 256 Schritten ist auf älteren iPhones langsam: lieber 256 Pixel Breite.
- **Schlitzbild-Breite:** höchstens 4096 Pixel, weil viele Geräte keine größeren Texturen haben. Bei längeren Videos wird nur jedes k-te Bild genommen.
- **Bilder von Künstlern:** Slit-Scan kennt man von Adam Magyar (Stainless, U-Bahnen), Zeitversatz von Golan Levins Sammlung „An Informal Catalogue of Slit-Scan Video Artworks“. Das ist ein guter Ort zum Nachsehen, was es schon gibt.

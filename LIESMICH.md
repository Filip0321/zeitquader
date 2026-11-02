# Zeitquader

Slit-Scan, Zeitversatz, der Video-Quader und Stepprinting in einem Projekt, weil alle dasselbe sind: verschiedene Schnitte durch einen Block aus Raum und Zeit. Ein eigenes Repo wie die anderen. Zum Einbauen ins Portfolio gibt es `merge-zeitquader.zip`.

**Einrichten**
- Inhalt in ein neues Repo `zeitquader` legen.
- `css/fonts/` aus dem Portfolio dazukopieren.
- Mit deinem lokalen Server ausliefern.

**Bedienung in Kürze**
1. Video öffnen (oder „Kamera“ für Schlitz und Zeitversatz live).
2. Ansicht wählen, Bereich einstellen, „Rechnen“.
3. Beim Quader: „kreisen“ (ziehen dreht, Mausrad zoomt) oder „fliegen“ (ziehen schaut, W A S D bewegt, Q und E runter und hoch, Umschalt schneller). Schnitt und Darstellung wirken sofort.
4. Export: „Video rendern“ macht beim Quader eine von drei Fahrten, beim Schlitz ein wachsendes Bild, bei Zeitversatz und Stepprinting das Video Bild für Bild. Eine eigene Fahrt nimmst du mit „Live aufnehmen“ auf, während du fliegst.

---

## 0. Neu in Runde 2: fliegen, Fahrten, mehr Bilder, Stepprinting

**Warum war die Kamerafahrt Pflicht?** Dafür gab es keinen tieferen Grund. Ich hatte nur eine Fahrt gebaut, weil sie die einfachste ist. Jetzt gibt es unter „Fahrt“ drei:
- **einmal herum:** wie bisher.
- **durch die Zeit fliegen:** Die Kamera fliegt vom ältesten zum neuesten Bild mitten durch den Block, den Blick nach vorn. Vor dir liegt die Zukunft des Videos, hinter dir die Vergangenheit.
- **Schnitt fährt durch:** Die Kamera steht, wie du sie gerade hast, und die Schnittfläche wandert einmal durch den Block. Ist der Schnitt aus, nimmt er „längs, senkrecht“, also einen wandernden Schlitz.

Eine Fahrt nach deinem Geschmack nimmst du mit **„Live aufnehmen“** auf: aufnehmen, fliegen, anhalten.

**Warum konnte man nicht hinein?** Auch hier gab es keinen Grund im Verfahren. Der Shader konnte das schon immer: Steht das Auge im Block, beginnt der Strahl bei 0 (`rein = max(…, 0.0)`). Nur die Kamera kreiste und hielt mindestens Abstand 1. Jetzt gibt es **„Kamera: fliegen“** (`flug()` in `kamera.js`):
- Die Kamera hat eine Stelle `pos` und zwei Winkel: `gier` (links und rechts) und `nick` (hoch und runter). Daraus wird die Blickrichtung `vorn(gier, nick)`.
- **W/S** bewegen entlang dieser Richtung. **A/D** bewegen entlang „rechts“: das ist das Kreuzprodukt aus Blickrichtung und „oben“, also immer waagerecht. **Q/E** bewegen senkrecht.
- Gesteuert wird pro Bild mit `requestAnimationFrame` und der vergangenen Zeit `dt`. Deshalb fliegt man auf jedem Rechner gleich schnell, egal wie viele Bilder pro Sekunde er schafft.
- Beim Wechsel startet die Flugkamera genau dort, wo die Kreiskamera stand (`kreisZuFlug`), damit das Bild nicht springt.
- Am Handy gibt es zwei Knöpfe „vor“ und „zurück“, ziehen schaut.
- Das ist dieselbe Kamera, die du später für Fraktale und Punktwolken brauchst.

**Warum höchstens 512 Pixel Breite und 256 Bilder?** Das waren meine vorsichtigen Grenzen, nicht die des Geräts:
- **256 Bilder** ist die Zahl, die WebGL2 mindestens garantiert (`MAX_ARRAY_TEXTURE_LAYERS`). Die meisten Desktop-Grafikkarten können 2048. Jetzt geht der Regler bis 1024, und `/lib/ringpuffer.js` nimmt, was Gerät und Speicherbudget (1,2 GB am Rechner, 300 MB am Handy) erlauben. Passen weniger, steht das in der Meldung.
- **Auflösung** jetzt bis 1024 breit. Rechnung: 1024 × 576 Pixel × 4 Byte = 2,4 MB pro Bild, ins Budget passen also etwa 500 Bilder.
- **Der eigentliche Preis ist die Rechenzeit:** Jeder Bildpunkt der Anzeige macht pro Schritt zwei Texturzugriffe. Mehr Bilder brauchen mehr Schritte, sonst überspringt der Strahl Bilder. Darum gilt jetzt `schritte` = anderthalb mal so viele wie Bilder, zwischen 256 und 1024. Während du ziehst oder fliegst, ist es die Hälfte, damit es flüssig bleibt. Lässt du los, wird einmal sauber gezeichnet. Die Schleife im Shader hat eine feste Obergrenze (1024) und bricht früher ab (`if (i >= schritte) break;`), weil manche Grafiktreiber nur Schleifen mit fester Länge übersetzen.

### Stepprinting (`BELICHTEN_SHADER`, `UEBERBLENDEN_SHADER`)

Den Look kennt man aus Wong Kar-wais *Chungking Express* und *Fallen Angels* (Kamera: Christopher Doyle). Gedreht wurde mit wenigen Bildern pro Sekunde und langer Belichtung, und im Kopierwerk wurde jedes Bild mehrmals hintereinander kopiert. Das Ergebnis: Die Bewegung stottert, und jede Stufe ist verwischt.

Digital in zwei Schritten:
1. **Belichten:** Alle „Schritt“ Bilder entsteht eine neue Aufnahme. Sie ist der Mittelwert der letzten „Belichtung“ Bilder im Ring, wie ein Verschluss, der so lange offen war. Was stillsteht, bleibt scharf, was sich bewegt, verwischt.
2. **Zeigen:** Zwischen zwei Aufnahmen steht das Bild still. Das ist das Stottern, also das eigentliche Step-Printing. Mit „Überblenden“ geht es weich von der vorigen Aufnahme in die neue über, das wirkt eher wie ein Traum als wie ein Stottern.

Die Aufnahmen liegen in zwei Texturen (`aufnahmen` in `gpu.js`). Bei jeder neuen Aufnahme werden sie vertauscht, damit die alte zum Überblenden noch da ist. Geht live mit der Kamera und mit Ton im Export.

Gute Werte zum Anfangen: Schritt 3 und Belichtung 3 (wie 10 Bilder pro Sekunde gedreht, mit offenem Verschluss); Schritt 4 und Belichtung 12 für sehr lange Schlieren.

---

## 1. Ein Video ist ein Quader

Leg alle Bilder eines Videos hintereinander wie Scheiben Brot. Dann hast du einen Block: x und y sind das Bild, die dritte Richtung ist die Zeit t. Ein normales Video zeigt diesen Block Scheibe für Scheibe, also immer einen Schnitt bei t = jetzt.

Jeder **andere Schnitt** ist ein anderes Bild:

| Schnitt | Ergebnis | Ansicht |
|---|---|---|
| t = fest | ein normales Bild | (Quader, Schnitt „quer“) |
| x = fest | aus jedem Bild dieselbe Spalte, nebeneinander: **Slit-Scan / Zielfoto** | Schlitz |
| eine schräge oder gekrümmte Fläche | jedes Pixel aus einer anderen Zeit: **Zeitversatz** | Zeitversatz |
| t als Treppe: jede Stufe ein Stück lang dieselbe Zeit, dafür etwas dick (über mehrere Bilder gemittelt) | **Stepprinting** | Stepprinting |
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
- *Rot, Grün, Blau versetzt:* der **Harris-Verschluss** (Robert Harris, Kodak, 1970er). Er hatte drei Farbfilter nacheinander vor dem Objektiv: Rot kommt von jetzt, Grün von „Tiefe“/2 Bilder früher, Blau von „Tiefe“ Bilder früher. Was stillsteht, ist in allen drei Kanälen gleich und bleibt farbtreu. Was sich bewegt, zerfällt in drei farbige Abbilder (Wasser, Wolken, Menschen). Live mit der Kamera am schönsten. Später lässt sich der Versatz audioreaktiv machen: Bass schiebt die Kanäle auseinander.

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
- Dazwischen 256 bis 1024 Schritte (anderthalb mal so viele wie Bilder im Block). An jedem wird aus dem Textur-Array gelesen, zwischen zwei Bildern gemischt. Es gibt also keine Lamellen, die Zeit ist durchgehend.

**Ergibt es Sinn?** Ehrlich: Als Bild ist der Quader schwerer lesbar als ein Schlitz. Man sieht vor allem die Ränder (die Außenseiten *sind* Schlitzbilder) und ein Leuchten im Innern. Stark wird er erst durch die Darstellung:

| Darstellung | was passiert | gut für |
|---|---|---|
| **nur Bewegtes** | Durchsichtig ist alles, was dem Median gleicht (wie beim Hintergrund-Abzug der Chronofotografie). Was sich bewegt, wird zu einer festen „Röhre“ durch die Zeit. | Personen, Autos, Vögel vor ruhigem Hintergrund. Das ist die eigentliche Idee. |
| **leuchten** | Alles wird nur aufaddiert, am Ende `1 - exp(-summe)` (damit es nicht ausbrennt) | Lichter in der Nacht, Feuerwerk |
| **Nebel** | alles ein wenig undurchsichtig, vorn verdeckt hinten | um den Block als Ding zu zeigen |

**Grenzen:** Alle Bilder müssen gleichzeitig auf der Grafikkarte liegen: 96 Bilder in 384 × 216 Pixeln sind etwa 30 MB, 512 Bilder in 1024 × 576 schon 1,2 GB. Darum gibt es den Quader nur mit Datei, nicht live, und nicht in voller Auflösung (Abschnitt 0). Für ein großes Standbild ist Blender besser. Was der Browser bietet: Man dreht in Echtzeit, schneidet auf und probiert Schwellen aus.

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
| `js/kamera.js` | Kreis- und Flugkamera → `auge` und `zurueck` |
| `lib/ringpuffer.js`, `lib/video-quelle.js` | gemeinsam mit Chronofotografie und Datamosh |

---

## 6. Ehrlich

- **Kamera live** geht für Schlitz und Zeitversatz. Beim Quader müsste man alle Bilder neu hochladen, sobald eins dazukommt. Machbar wäre ein wandernder Quader (Ringpuffer, Ebene 0 springt), aber das habe ich gelassen.
- **Handys:** Der Ringpuffer ist auf dem Handy auf 300 MB begrenzt, der Zeitversatz hat dann weniger Tiefe. Der Quader ist auf älteren iPhones langsam: lieber 256 Pixel Breite und wenige Bilder.
- **Schlitzbild-Breite:** höchstens 4096 Pixel, weil viele Geräte keine größeren Texturen haben. Bei längeren Videos wird nur jedes k-te Bild genommen.
- **Bilder von Künstlern:** Slit-Scan kennt man von Adam Magyar (Stainless, U-Bahnen), Zeitversatz von Golan Levins Sammlung „An Informal Catalogue of Slit-Scan Video Artworks“. Das ist ein guter Ort zum Nachsehen, was es schon gibt.

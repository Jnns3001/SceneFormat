
AI-Assisted 60%: Im Dialog mit KI ausgearbeitet; Entscheidungen von Jannis Mayer, Ausformulierung durch KI.
Code-Owner: Jannis Mayer

# Szenen-Austauschformat – Spezifikation Version 1

**Status:** Entwurf vom 2026-11-16, herausgegeben von der Gruppe Rusttracer.
Rückmeldungen der anderen Gruppen fließen in diese Spezifikation ein.

## 1. Zweck und Umfang

Das Format beschreibt eine Szene so, dass sie gespeichert, wieder geladen und zwischen
unabhängig entwickelten Raytracern ausgetauscht werden kann. Verbindlich ist allein diese
Spezifikation, nicht eine bestimmte Implementierung.

**Leser müssen NICHT das gesamte Format unterstützen**; Jeder Leser darf beliebig viele Felder ignorieren!
Jeder Leser darf die Daten aus dem Format intern frei verwenden und umwandeln.
**Schreiber müssen gültiges Dokument und Paket erzeugen;** Auch ein Schreiber muss nicht das gesamte Format unterstützen.
Jene Felder und Typen die aber genutzt werden MÜSSEN gemäß dem Format valide sein.
Externe Dateien im mitgelieferten [Paket](#11-paket) (z.B. .OBJ) MÜSSEN valide sein und nur [lokale Pfade](#10-pfade) enthalten .

Eine Szene besteht aus:

- Kameras: beliebig viele Ansichten derselben Szene,
- Objekten: Kugeln, Dreiecke und externe Modelle (Wavefront OBJ/MTL),
- Lichtern: Punkt-, Richtungs- und Umgebungslicht,
- einem Hintergrund: Farbe, Verlauf oder Umgebungsbild.

Nicht Teil des Formats sind MTL/OBJ Format, Auflösung, Renderqualität (Abtastungen, Strahltiefe),
programmspezifischer Zustand wie Bedienoberflächen und Beschleunigungsstrukturen.

Datenmodell als Diagramm: [Szenenformat-Datenmodell](../blob/main/plantUML/Szenenformat-Datenmodell.puml)

## 2. Begriffe

- **MUSS** / **DARF NICHT**: verbindlich.
- **SOLL**: Empfehlung; Abweichen nur mit gutem Grund.
- **DARF**: freigestellt.
- **Szenendokument**: JSON-Datei nach dieser Spezifikation.
- **Paket**: Ordner mit Szenendokument und allen referenzierten Dateien ([Abschnitt 11](#11-paket)).
- **Schreiber**: Programm, das ein Szenendokument oder Paket erzeugt.
- **Leser**: Programm, das ein Szenendokument lädt.

## 3. Konventionen und Datentypen

| Thema                      | Festlegung                                                                                    |
| -------------------------- | --------------------------------------------------------------------------------------------- |
| Koordinatensystem          | rechtshändig, +y zeigt nach oben                                                             |
| Einheiten                  | frei; es wird nichts umgerechnet                                                              |
| Winkel                     | Grad                                                                                          |
| Farben                     | lineares RGB, keine sRGB-Kodierung; Werte über 1 sind erlaubt                                |
| Vorderseite eines Dreiecks | Ecken gegen den Uhrzeigersinn, von vorne gesehen; Normale `normalize((b − a) × (c − a))` |
| Zahlen                     | JSON-Zahlen; alle Werte endlich (JSON kennt kein NaN oder ±∞)                               |
| Kodierung                  | UTF-8                                                                                         |
| Feldnamen und Typwerte     | Englisch, `snake_case`                                                                      |

Die Tabellen verwenden folgende Datentypen:

| Typ         | JSON                                | Bedeutung                                                                    |
| ----------- | ----------------------------------- | ---------------------------------------------------------------------------- |
| `integer` | Ganzzahl                              | –                                                                           |
| `number`  | Zahl                                | –                                                                           |
| `string`  | String                              | –                                                                           |
| `vec3`    | Array aus 3 Zahlen `[x, y, z]`    | Punkt oder Richtung                                                          |
| `quat`    | Array aus 4 Zahlen `[x, y, z, w]` | Rotation als Quaternion; Leser normieren sie; `[0, 0, 0, 0]` ist ungültig |
| `color`   | Array aus 3 Zahlen `[r, g, b]`    | Farbe; 0 ≤ Komponenten ≤ 1 (>1 erlaubt für sRGB oder HDR                        |
| `path`    | String                              | Dateipfad nach [Abschnitt 10](#10-pfade)                                      |

Weicht die interne Konvention eines Lesers ab, rechnet er beim Laden um.

## 4. Dokument

Wurzel ist ein JSON-Objekt.

| Feld           | Typ                           | Pflicht | Bedeutung                                      |
| -------------- | ----------------------------- | ------- | ---------------------------------------------- |
| `version`    | `integer`                   | ja      | Hauptversion des Formats, hier `1`           |
| `cameras`    | Array von [Kamera](#5-kameras) | nein    | Ansichten; die erste ist die aktive            |
| `objects`    | Array von [Objekt](#6-objekte) | nein    | Inhalt der Szene; fehlt es, ist die Szene leer |
| `lights`     | Array von [Licht](#8-lichter)  | nein    | Lichtquellen                                   |
| `background` | [Hintergrund](#9-hintergrund)  | nein    | fehlt er, wählt der Leser seinen eigenen      |

Fehlt `cameras` oder ist das Array leer, MUSS der Leser die
[Standardkamera](#5-kameras) verwenden.

Fehlt `lights`, wählt der Leser seine eigene Beleuchtung. Ein leeres Array bedeutet
ausdrücklich: keine Lichtquellen.

Kleinstes gültiges Dokument (Standardkamera, leere Szene):

```json
{ "version": 1 }
```

## 5. Kameras

Lochkamera; das Seitenverhältnis ergibt sich aus der Auflösung des Renderers.

| Feld             | Typ        | Pflicht | Standard       | Gültige Werte                                             |
| ---------------- | ---------- | ------- | -------------- | ---------------------------------------------------------- |
| `name`         | `string` | nein    | –             | nur zur Anzeige                                            |
| `look_from`    | `vec3`   | nein    | `[0, 0, 0]`  | Position der Kamera                                        |
| `look_at`      | `vec3`   | nein    | `[0, 0, -1]` | ungleich `look_from`; der Abstand ändert das Bild nicht |
| `vup`          | `vec3`   | nein    | `[0, 1, 0]`  | ungleich null, nicht parallel zur Blickrichtung            |
| `vfov_degrees` | `number` | nein    | `90`         | vertikales Sichtfeld, `0 < x < 180`                      |

Die Reihenfolge der Kameras MUSS erhalten bleiben. Die erste Kamera ist beim Laden aktiv.

**Standardkamera:** Eine Kamera, deren Felder alle ihren Standardwert haben. Sie steht im
Ursprung, blickt entlang −z, +x liegt rechts. Sie gilt, wenn `cameras` fehlt oder leer ist.

## 6. Objekte

Jedes Objekt ist ein JSON-Objekt mit dem Pflichtfeld `type`. Alle Objekttypen haben das
optionale Feld `name` (`string`, nur zur Anzeige; Namen sind nicht eindeutig und keine
Referenz). Die Reihenfolge der Objekte MUSS beim Laden und Speichern erhalten bleiben.

### 6.1 `sphere`

| Feld         | Typ                    | Pflicht | Standard | Gültige Werte                                                |
| ------------ | ---------------------- | ------- | -------- | ------------------------------------------------------------- |
| `center`   | `vec3`               | ja      | –       | –                                                            |
| `radius`   | `number`             | ja      | –       | ungleich 0; ein negativer Radius kehrt die Normale nach innen |
| `material` | [Material](#7-material) | nein    | –       | –                                                            |

### 6.2 `triangle`

| Feld         | Typ                    | Pflicht | Standard | Gültige Werte                                                                                     |
| ------------ | ---------------------- | ------- | -------- | -------------------------------------------------------------------------------------------------- |
| `vertices` | Array aus 3 `vec3`   | ja      | –       | nicht entartet (Ecken nicht gleich und nicht kollinear); die Reihenfolge legt die Vorderseite fest |
| `material` | [Material](#7-material) | nein    | –       | –                                                                                                 |

Die Ecken liegen in Weltkoordinaten. Ob die Rückseite sichtbar ist, entscheidet der Leser.

### 6.3 `model`

| Feld          | Typ                                 | Pflicht | Standard   | Gültige Werte      |
| ------------- | ----------------------------------- | ------- | ---------- | ------------------- |
| `path`      | `path`                            | ja      | –         | Wavefront-OBJ-Datei |
| `transform` | [Transformation](#64-transformation) | nein    | Identität | –                  |

Die OBJ-Datei liegt in lokalen Koordinaten mit derselben Achsenkonvention wie die Szene
([Abschnitt 3](#3-konventionen-und-datentypen)). Materialien kommen aus den MTL-Dateien, auf
die die OBJ-Datei mit `mtllib` verweist, Texturen aus deren `map_*`-Einträgen. Mehrere
`model`-Objekte DÜRFEN dieselbe Datei verwenden; jedes ist eine eigene Platzierung. Leser
DÜRFEN die Geometrie dabei einmal laden und teilen.

### 6.4 Transformation

Bildet einen lokalen Punkt `p` auf `translation + rotation · (scale ⊙ p)` ab, als Matrix
`T · R · S`: erst skalieren, dann rotieren, dann verschieben.

| Feld            | Typ      | Pflicht | Standard         | Gültige Werte                                      |
| --------------- | -------- | ------- | ---------------- | --------------------------------------------------- |
| `translation` | `vec3` | nein    | `[0, 0, 0]`    | –                                                  |
| `rotation`    | `quat` | nein    | `[0, 0, 0, 1]` | –                                                  |
| `scale`       | `vec3` | nein    | `[1, 1, 1]`    | jede Komponente ungleich 0; negative Werte spiegeln |

Bei einer ungeraden Anzahl negativer Skalierungskomponenten (Spiegelung) kehrt sich die
Vorderseite um; Leser MÜSSEN das berücksichtigen.

## 7. Material

Phong-Material für Kugeln und Dreiecke; die Felder entsprechen den gleichnamigen MTL-Feldern.
Alle Felder sind optional. Fehlt `material` ganz, verwendet der Leser sein Standardmaterial.

| Feld          | MTL    | Typ        | Standard            | Gültige Werte  |
| ------------- | ------ | ---------- | ------------------- | --------------- |
| `ambient`   | `Ka` | `color`  | `[0, 0, 0]`       | –              |
| `diffuse`   | `Kd` | `color`  | `[0.8, 0.8, 0.8]` | –              |
| `specular`  | `Ks` | `color`  | `[0, 0, 0]`       | –              |
| `shininess` | `Ns` | `number` | `0`               | ≥ 0            |
| `opacity`   | `d`  | `number` | `1`               | `0 ≤ x ≤ 1` |

Ein Material ohne Felder (`{}`) ist ein diffuses Grau. Modelle verwenden die Materialien
ihrer MTL-Dateien.

## 8. Lichter

Lichter stehen in einer eigenen Liste `lights`, nicht unter `objects`: Sie sind unsichtbar
und werden von Strahlen nicht getroffen. Jedes Licht ist ein JSON-Objekt mit dem Pflichtfeld
`type` und dem optionalen Feld `name` (`string`, nur zur Anzeige). Die Reihenfolge MUSS
erhalten bleiben.

Alle Lichttypen haben:

| Feld          | Typ        | Pflicht | Standard      | Gültige Werte |
| ------------- | ---------- | ------- | ------------- | -------------- |
| `color`     | `color`  | nein    | `[1, 1, 1]` | –             |
| `intensity` | `number` | nein    | `1`         | ≥ 0           |

### 8.1 `point`

Punktlicht, strahlt in alle Richtungen.

| Feld         | Typ      | Pflicht | Standard | Gültige Werte |
| ------------ | -------- | ------- | -------- | -------------- |
| `position` | `vec3` | ja      | –       | –             |

Im Abstand `d` trifft das Licht mit `color · intensity / d²` ein.

### 8.2 `directional`

Unendlich weit entferntes Licht, etwa die Sonne.

| Feld          | Typ      | Pflicht | Standard | Gültige Werte                                    |
| ------------- | -------- | ------- | -------- | ------------------------------------------------- |
| `direction` | `vec3` | ja      | –       | ungleich null; Richtung, in die das Licht strahlt |

Das Licht trifft überall mit `color · intensity` ein.

### 8.3 `ambient`

Konstanter Umgebungsanteil ohne Richtung und ohne Schatten; wirkt mit dem `ambient`-Anteil
der Materialien. Keine weiteren Felder.

### 8.4 Beispiel: Phong-Auswertung (informativ)

Dieser Abschnitt ist nicht verbindlich. Eine Phong-Auswertung an einem Punkt mit Normale `n`,
Blickrichtung `v` und Material `(Ka, Kd, Ks, Ns)` kann so aussehen:

```text
color = Σ ambient lights  Ka · E
      + Σ other lights    E · (Kd · max(0, n·l) + Ks · max(0, r·v)^Ns)
```

Dabei ist `E` das eintreffende Licht nach 8.1 bis 8.3, `l` die normierte Richtung vom Punkt
zum Licht und `r` die an `n` gespiegelte Lichtrichtung. Ob Schatten berechnet werden und wie
ein Leser Phong auf ein anderes Lichtmodell abbildet, entscheidet er selbst.

## 9. Hintergrund

Legt fest, was ein Strahl sieht, der kein Objekt trifft. Ob der Hintergrund die Objekte
zusätzlich beleuchtet, entscheidet der Leser. Pflichtfeld ist `type`.

### 9.1 `solid`

| Feld      | Typ       | Pflicht | Standard |
| --------- | --------- | ------- | -------- |
| `color` | `color` | ja      | –       |

Eine Farbe in allen Richtungen.

### 9.2 `gradient`

| Feld       | Typ       | Pflicht | Standard |
| ---------- | --------- | ------- | -------- |
| `bottom` | `color` | ja      | –       |
| `top`    | `color` | ja      | –       |

Für die normierte Strahlrichtung `d` gilt `t = 0.5 · (d.y + 1)` und
`color = (1 − t) · bottom + t · top`.

Beispiel für einen hellen Himmel: `{ "type": "gradient", "bottom": [1, 1, 1], "top": [0.5, 0.7, 1] }`.

### 9.3 `environment`

Umgebungsbild, das die Szene in allen Richtungen umgibt.

| Feld           | Typ        | Pflicht | Standard              | Gültige Werte                 |
| -------------- | ---------- | ------- | --------------------- | ------------------------------ |
| `path`       | `path`   | ja      | –                    | Bilddatei                      |
| `projection` | `string` | nein    | `"equirectangular"` | siehe unten                    |
| `rotation`   | `quat`   | nein    | `[0, 0, 0, 1]`      | dreht das Bild um die Szene    |
| `intensity`  | `number` | nein    | `1`                 | ≥ 0; Faktor auf die Bildfarbe |

**`equirectangular`** (Längen-/Breitengrad-Abbildung, Seitenverhältnis 2:1): Für eine
Strahlrichtung `ray_dir` wird zuerst die Rotation herausgerechnet,
`d = normalize(rotation⁻¹ · ray_dir)`. Dann gilt

```text
u = 0.5 + atan2(d.x, −d.z) / (2π)
v = arccos(d.y) / π
```

`u` läuft von 0 (linker Bildrand) bis 1 (rechter Bildrand), `v` von 0 (oberste Bildzeile)
bis 1 (unterste Bildzeile).

Die Bildmitte liegt also in Richtung −z, +x rechts davon, +y am oberen Bildrand. Der
Übergang bei `u = 0` und `u = 1` (Richtung +z) ist die Naht.

Schreiber SOLLEN PNG (sRGB, wird beim Laden linearisiert) oder Radiance HDR (`.hdr`, linear)
verwenden; Leser SOLLEN beide unterstützen.

Andere Werte für `projection` DÜRFEN verwendet werden; ihre Bedeutung legt diese Version nicht
fest. Ein Leser, der eine Projektion nicht kennt, behandelt den Hintergrund wie einen fehlenden.

## 10. Pfade

- Pfade sind Strings. Relative Pfade beziehen sich auf den Ordner des Szenendokuments, nicht
  auf das Arbeitsverzeichnis.
- Leser MÜSSEN relative Pfade mit `/` als Trenner unterstützen und DÜRFEN zusätzlich absolute
  Pfade und `..` akzeptieren.
- Innerhalb eines [Pakets](#11-paket) sind nur relative Pfade mit `/` ohne `..` erlaubt.

## 11. Paket

Für den Austausch wird eine Szene als Paket weitergegeben:

```text
<scene>/
├── scene.json
├── models/
│   └── <model>/
│       ├── <model>.obj
│       ├── <model>.mtl
│       └── ...
└── environment/
    └── <image>
```

- Das Szenendokument heißt `scene.json` und liegt im Wurzelordner des Pakets.
- Alle referenzierten Dateien liegen im Paket. Jede Datei, auf die das Szenendokument, eine
  OBJ- oder eine MTL-Datei verweist, ist über einen relativen Pfad ohne `..` erreichbar.
- OBJ-, MTL- und Bilddateien werden unverändert übernommen; Texturen behalten ihre relative
  Lage zur MTL-Datei.
- Jedes Modell SOLL in einem eigenen Unterordner von `models/` liegen, damit gleich benannte
  Dateien verschiedener Modelle nicht kollidieren. Die Ordnernamen `models/` und
  `environment/` sind eine Empfehlung; Leser folgen nur den Pfaden im Dokument.
- Lässt sich eine Szene so nicht verpacken, weil eine OBJ- oder MTL-Datei eine Datei
  außerhalb ihres Ordners referenziert (`..` oder absolut), MUSS der Schreiber den Export
  ablehnen. Er DARF Referenzen nicht umschreiben und kein unvollständiges Paket erzeugen.

**Ordner und ZIP:** Ein Paket ist ein Ordner. Schreiber MÜSSEN Pakete als Ordner erzeugen
können und DÜRFEN sie zusätzlich als ZIP-Archiv (`.zip`) anbieten. Leser MÜSSEN Ordner
akzeptieren und DÜRFEN ZIP-Archive akzeptieren. In einem ZIP-Archiv liegt der Inhalt des
Wurzelordners, also `scene.json`, direkt in der Wurzel des Archivs.

## 12. Versionen und Erweiterungen

- `version` ist die Hauptversion. Sie steigt nur, wenn sich die Bedeutung bestehender Felder
  ändert oder ein Leser der alten Version die Szene sonst falsch darstellen würde.
- Neue optionale Felder und neue Objekt-, Licht-, Hintergrund- oder Projektionstypen erhöhen
  die Version nicht.
- Leser DÜRFEN unbekannte Felder und Felder, die sie nicht unterstützen, ignorieren.
- Leser SOLLEN Objekte und Lichter mit unbekanntem `type` überspringen und den Nutzer darauf
  hinweisen. Einen Hintergrund mit unbekanntem `type` behandeln sie wie einen fehlenden.
- Leser SOLLEN Dokumente mit höherer als der unterstützten Version ablehnen.

## 13. Gültigkeit und Fehlerbehandlung

- Schreiber MÜSSEN gültige Dokumente und Pakete garantieren, also alle Pflichtfelder und
  Wertebereiche dieser Spezifikation einhalten.
- Leser DÜRFEN unbekannte oder für sie ungenutzte Felder ignorieren, etwa `material` in einem
  Renderer ohne Materialien ([Abschnitt 12](#12-versionen-und-erweiterungen)).
- Wie ein Leser mit ungültigen Dokumenten oder Werten umgeht (ablehnen, Objekt überspringen,
  Wert korrigieren, warnen), entscheidet jede Implementierung selbst.
- Schreiber SOLLEN optionale Felder mit Standardwert ausdrücklich ausschreiben, damit
  Dokumente auch ohne Kenntnis der Standardwerte lesbar bleiben.

## 14. Beispiel

`scene.json` eines Pakets:

```json
{
  "version": 1,
  "cameras": [
    {
      "name": "front",
      "look_from": [0, 0, 0],
      "look_at": [0, 0, -1],
      "vup": [0, 1, 0],
      "vfov_degrees": 90
    },
    {
      "name": "side",
      "look_from": [4, 1, -2],
      "look_at": [0, 0, -2],
      "vup": [0, 1, 0],
      "vfov_degrees": 60
    }
  ],
  "objects": [
    {
      "type": "sphere",
      "name": "floor",
      "center": [0, -1001, 0],
      "radius": 1000,
      "material": { "diffuse": [0.5, 0.5, 0.5] }
    },
    {
      "type": "sphere",
      "center": [1.5, 0.2, -1.5],
      "radius": 0.5,
      "material": { "diffuse": [0.8, 0.1, 0.1], "specular": [1, 1, 1], "shininess": 64 }
    },
    {
      "type": "triangle",
      "vertices": [[-2.4, 0.9, -2], [-1, 0.9, -2], [-1.7, 1.8, -2]]
    },
    {
      "type": "model",
      "name": "pyramid",
      "path": "models/pyramid/pyramid.obj",
      "transform": {
        "translation": [2.1, 1.3, -2.15],
        "rotation": [0, 0, 0, 1],
        "scale": [1, 1, 1]
      }
    },
    {
      "type": "model",
      "name": "hanging pyramid",
      "path": "models/pyramid/pyramid.obj",
      "transform": {
        "translation": [-0.3, 1.6, -2.5],
        "rotation": [1, 0, 0, 0],
        "scale": [1, 1, 1]
      }
    }
  ],
  "lights": [
    { "type": "ambient", "color": [1, 1, 1], "intensity": 0.1 },
    { "type": "point", "name": "lamp", "position": [0, 3, 0], "color": [1, 1, 1], "intensity": 20 },
    { "type": "directional", "name": "sun", "direction": [-1, -1, -1], "color": [1, 0.95, 0.9], "intensity": 1 }
  ],
  "background": {
    "type": "gradient",
    "bottom": [1, 1, 1],
    "top": [0.5, 0.7, 1]
  }
}
```

Paketinhalt dazu:

```text
scene/
├── scene.json
├── models/
│   └── pyramid/
│       ├── pyramid.obj
│       └── pyramid.mtl
```

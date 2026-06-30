# GPGPU-Implementierung — Entwickler-Notizen

Diese Datei sammelt die Architektur-Begründungen ("warum", nicht "was") für
`index.html`, die früher als lange Inline-Kommentare im Code standen. Der
Code selbst bleibt knapp kommentiert; Hintergrund und Abwägungen stehen hier.

## Grundprinzip

Der komplette Partikelzustand (Position, Geschwindigkeit, Alter) lebt in
Float-Texturen auf der GPU (Ping-Pong zwischen zwei Texturen, siehe
`initWebGL()`), nicht mehr in JS-Typed-Arrays auf der CPU. Es gibt keinen
Canvas2D-Fallback mehr — fehlen die nötigen GPU-Fähigkeiten, wird das laut
gemeldet statt leise auf eine langsamere Variante umzuschalten.

Flow-Feld, Text-Gravitationsfeld und Farbpalette werden jeweils nach jeder
Neuberechnung als kleine Float-Texturen hochgeladen, damit die Compute-Shader
(Lifecycle-/Physik-Pass) sie per `texture()` sampeln können. Die Paletten-
Textur ist bewusst klein (32×1, 128 Bytes UNSIGNED_BYTE) und wird jeden
Frame neu hochgeladen, weil die Farb-Animation sie laufend überblendet.

## Partikelanzahl: MAX, Kalibrator, Einblend-Rampe

- `MAX` bestimmt nur noch `TEX_DIM` (die feste Größe der Zustands-Texturen),
  keine CPU-Arrays mehr. Muss vor `COUNT_CALIB_MAX` deklariert sein, da der
  Kalibrator bis an `MAX` heran hochtasten darf.
- `COUNT_CALIB_MAX` ist bewusst niedriger als `MAX` (die reine Textur-/
  Slider-Obergrenze): der Kalibrator soll automatisch höchstens bis 1 Mio.
  hochtasten, alles darüber ist manuell (Slider/Presets).
- Der Kalibrator tastet `S.count` beim Start in `COUNT_CALIB_STEP`-Schritten
  von `COUNT_CALIB_START` nach oben, solange die FPS nahe 60 bleiben — findet
  so automatisch die für die jeweilige Hardware höchstmögliche Partikelzahl,
  statt eines für alle Geräte fest verdrahteten Werts. Ziel: durchgehend
  ≥50fps, die 10%-Lows nicht darunter. Erlischt dauerhaft, sobald `S.count`
  per Nutzeraktion verändert wird (Slider, Preset, Import, Surprise Me) —
  siehe `_countCalibStop()`.
- Er misst bewusst die rohe, ungeglättete Pro-Frame-FPS (`instFps`) statt
  einer geglätteten EMA: ein EMA würde kurze Ruckler genau wegmitteln, also
  die Sache verschleiern, die hier erkannt werden soll.
- Ein nicht bestandenes Fenster ist evtl. nur ein kurzzeitiger Hänger (z. B.
  Tab-Wechsel, GC-Pause) — erst nach `COUNT_CALIB_MAX_FAILS` aufeinander-
  folgenden Fehlversuchen auf derselben Stufe wird wirklich zurückgerudert.
- **Einblend-Rampe (`_countRamp`):** wenn der Kalibrator `S.count` erhöht,
  sollen die neu hinzukommenden Partikel nicht alle schlagartig im selben
  Frame auftauchen (sie starten ohnehin alle an Position (0,0), siehe
  Seed-Kommentar in `initWebGL()` — ein plötzlicher Sprung um
  `COUNT_CALIB_STEP` wirkt dadurch wie ein kleiner Partikel-"Ausbruch" aus
  einer Ecke). `_countRamp` blendet die für Compute/Render aktive
  Partikelzahl stattdessen über `COUNT_RAMP_DUR` ms weich von "from" auf
  "to" ein/aus. Nur für automatische Kalibrator-Änderungen aktiv — direkte
  Nutzeraktionen (Slider, Presets, Import, Surprise Me) reagieren über
  `_countCalibStop()` weiterhin sofort, ohne Rampe.
- Pro Frame werden in `animate()` nur so viele Texel-Zeilen wie für
  `S.count` tatsächlich gebraucht neu berechnet (Partikel-Index → Texel ist
  row-major, siehe `uvData` in `initWebGL()`) — das entkoppelt die
  Pro-Frame-GPU-Kosten von `MAX` (der reinen Textur-Kapazität) und macht sie
  wieder von `S.count` abhängig, wie im CPU-Original. Sonst würde jede
  `MAX`-Erhöhung die Rechenlast für alle Nutzer erhöhen, unabhängig von der
  gewählten Partikelanzahl. `_effCountNow()` statt `S.count` direkt sorgt
  dafür, dass automatische Kalibrator-Erhöhungen weich über `COUNT_RAMP_DUR`
  einblenden; bei Nutzeraktionen ist die Rampe bereits über
  `_countCalibStop()` auf den neuen Wert gesnapped, reagiert also sofort.

## Kein Performance-Governor / kein Staffeln (physK)

Im CPU-Original gab es einen adaptiven Performance-Governor, der nur einen
Bruchteil der Partikel pro Frame aktualisierte (`physK`). Auf der GPU
entfällt das: die GPU aktualisiert jeden Frame ALLE Partikel in der
State-Textur, unabhängig von `S.count` — es gibt genug Spielraum, kein
Staffelungsfaktor nötig.

## Force-Cutoff-Radius

Jenseits einer gewissen Distanz ist `mag = stärke/(d²+softening)` visuell
nicht mehr wahrnehmbar (≤ `FORCE_EPS` px/frame²) → `sqrt()` + Kraftberechnung
für Partikel jenseits dieser Distanz überspringen. Größter Hebel bei mehreren
simultanen Kraftquellen (Komet ×8 + Auto-Klick + Feuerwerk), da diese sonst
jede `COUNT`-mal pro Frame ungeachtet der Distanz ein `sqrt()` ausführen
würden.

## GPGPU-Kraftliste

Klick/Auto-Klick/Feuerwerk/Komet-Pool werden zu einem festen Uniform-Array
zusammengefasst (max. 11 Slots: 1+1+1+8), statt wie im CPU-Original einzeln
im Partikel-Loop abgefragt zu werden — der Physik-Shader iteriert einmal über
`uForceCount` Einträge pro Partikel/Frame.

## Sin-freier Hash (Spawn-Position)

GPU-`sin()`-Implementierungen (v. a. ANGLE/D3D, manche Mobile-GPUs) verlieren
für große Argumente massiv an Genauigkeit, und
`dot(uv*1000+uRespawnSeed, vec2(127.1,311.7))` erzeugt genau solche großen
Argumente — das Ergebnis kollabiert dann auf eine Handvoll fester Werte, was
sich als "Partikel spawnen immer an denselben paar Stellen" zeigt. Die
Lifecycle-/Physik-Shader nutzen daher einen `fract()`-basierten Hash ohne
`sin()`, der alle Zwischenwerte klein hält, unabhängig von der
Eingabe-Größenordnung. `uRespawnSeed` wird zusätzlich mit Modulo gewrappt,
damit er bei sehr langer Laufzeit (Stunden) nicht unbegrenzt wächst.

Die gemeinsame Compute-Logik für Lifecycle- und Physik-Pass ist als String
einmal definiert und in beide Fragment-Shader eingebettet, damit beide
Passes garantiert identische Physik-/Respawn-Entscheidungen treffen (keine
Divergenz).

## Half-Float-Fallback

Für den seltenen Fall, dass eine GPU nur in Half-Float-Texturen rendern kann
(siehe `STATE_GLTYPE`-Fallback in `initWebGL()`), gibt es eine Funktion, die
ein einzelnes Float32 in sein Half-Float-Bitmuster umrechnet.

## Render-Pass-Viewport (uRowFrac)

Der Render-Vertex-Shader nutzt `uRowFrac` (= benötigte Texel-Zeilen /
`TEX_DIM`), um `vUv.y` auf den tatsächlich aktiven Bereich der
Zustands-Textur zu stauchen, passend zum mit
`gl.viewport(0,0,TEX_DIM,rowsNeeded)` verkleinerten Render-Ziel in
`animate()` — so bleibt die 1:1-Zuordnung Ziel-Pixel-Zeile ↔ Quell-Texel-Zeile
erhalten, statt den vollen 0..1-UV-Bereich in das kleinere Viewport zu
stauchen (was Zeilen vertikal verzerrt/vermischt hätte).

## Partikel-Seed-Reihenfolge

Die Seed-Daten beim Start replizieren bewusst dieselbe Reihenfolge/Formel wie
`spawnParticle(i,true)` im ursprünglichen CPU-Modell, und `initWebGL()` läuft
bewusst VOR `resize()` (W=H=0 zu diesem Zeitpunkt) — die Partikel starten alle
bei (0,0) und verteilen sich durch ihr zufälliges Alter natürlich über den
Bildschirm.

## FBM (Fractal Brownian Motion)

Zweite Noise-Oktave bei 2,1× Frequenz, 0,45× Amplitude. Erzeugt organischere,
mehrskalige Wirbelstrukturen; Kosten ca. 2× Noise-Aufrufe pro Update
(2880 Zellen × 8 statt 4 Aufrufe) — vernachlässigbar bei 3-Frame-Intervall.

## Text-Gravitationsfeld

Eigenes hochauflösendes Gitter (4× Flow-Field-Auflösung), damit dünne
Buchstabenstriche sicher erfasst werden — bei 64×45 (Flow-Field-Auflösung)
waren Zellen ~30px breit, zu grob für Text.

**Sanfter Abfall am Rand:** der Suchradius (`SR`) begrenzt, wie weit eine
Zelle nach dem nächsten Textpixel sucht. Statt jenseits von `SR` hart auf 0
zu springen, läuft die Zugkraft per Smoothstep weich gegen 0 aus, während sie
nah am Text unverändert voll bleibt — sonst entsteht am Rand ein sichtbarer
"Kasten" durch den abrupten Sprung auf 0.

## Komet (Schweifeffekt) — Pool statt Einzelobjekt

Virtuelle "Klickpunkte" wandern entlang leicht gekrümmter Bahnen über die
Leinwand (wie Kometen) und ziehen Partikel an, mit konfigurierbarer
Intensitätskurve über die Laufzeit. Im normalen Modus läuft immer höchstens
ein Komet gleichzeitig (einfacher periodischer Trigger); im Schauer-Modus
feuert ein Feuerwerk-artiger Burst mehrere Kometen mit kleinem Versatz ab,
die sich zeitlich überlappen können — daher ein Pool statt eines
Einzelobjekts.

## Musik-System

- **Limiter:** fängt Pegelspitzen ab, wenn viele Feuerwerk-Klicks/Sparkle-
  Grains gleichzeitig mit dem Drone-Layer summieren. Verhindert das hörbare
  Clipping/Übersteuern ("Stecker reinstecken"-Knacksen), das ohne Limiter bei
  dichten Feuerwerk-Bursts auftrat.
- **High-Bus** umgeht bewusst den Lowpass — sonst killt die energie-
  getrackte Cutoff (oft 180–500 Hz) jede Helligkeit im Mix. Trägt
  Pad/hohe Noten/Sparkle.
- **FX-Bus** für Feuerwerk-Booms ist ebenfalls ungefiltert, damit der
  Sub-Thump unabhängig von der aktuellen Energie-Cutoff-Automation immer
  voll durchkommt.
- **Mid/High-Pad:** zwei langsam "atmende" Lagen via sample-genauem LFO
  (Oszillator moduliert direkt den Gain-Wert) statt JS-Timing — macht aus dem
  Drone eine Komposition, die sich über die Zeit bewegt, statt einem
  einzigen durchgehenden Ton.
- Ambient-Noten mischen tiefe (Low-Bus) und hohe (High-Bus) Lagen statt nur
  einer, damit es sich wie eine Komposition aus verschiedenen Tönen anhört;
  `strength` (0..1, aus Feuerwerk-Klickstärke abgeleitet) hebt stärkere
  Klicks eine Oktave + etwas lauter, bleibt aber im ruhigen Rahmen.
- **Feuerwerk-"Platzen":** Sub-Bass-Thump (Pitch-Drop) + gefilterter
  Noise-Burst (Schockwelle, hoch→tief gewedelt) + kurzer metallischer
  Ring-Oberton ("Platte"-Echo), ersetzt eine frühere saubere Sinus-Note, die
  nicht nach einer Explosion klang.
- Mehr Partikel-Energie → dichtere Sparkle-Einsätze, damit sich der
  Glitzer-Layer tatsächlich an der Partikelströmung orientiert statt nur den
  Lowpass zu modulieren.

## Weiche Farbverläufe statt Bucket-Kanten

Ursprünglich wurde jeder Partikel auf einen von 16 diskreten Hue-„Buckets“
gemappt (`floor(t*16)`), kombiniert mit einer 32×1-Palette-Textur, die mit
NEAREST gesampelt wurde. Sobald sich ein kontinuierlicher Antriebswert
(v. a. `colorDrive='velocity'`, da Geschwindigkeit ständig leicht schwankt)
über eine Bucket-Grenze bewegte, sprang die Farbe abrupt zum Nachbar-Bucket —
sichtbar als harte Kante, bei Farbschema-Übergängen (`colorAnimOn`) zusätzlich
verstärkt, weil sich dort auch die Bucket-Farben selbst gerade veränderten.

Fix: die Palette-Textur ist jetzt 2D (`N_HUE × N_BRT` = 16×2 statt 32×1) und
wird mit LINEAR statt NEAREST gefiltert; der Vertex-Shader berechnet pro
Partikel einen stetigen Hue-Wert (`hueT`, kein `floor()`/Bucket-Index mehr)
und eine stetige Helligkeit (`brt`, per `smoothstep()` im Lifecycle-Pass statt
binärem 0/1-Sprung mit Hysterese). Beides zusammen als `vPalUV` an den
Fragment-Shader übergeben — die GPU interpoliert dadurch bilinear zwischen
benachbarten Palette-Einträgen in beiden Achsen (Farbton UND Helligkeit),
statt hart zwischen festen Werten zu springen. Das verbessert sowohl die
laufende Geschwindigkeits-Färbung als auch jeden Übergang zwischen zwei
Farbschemata (inklusive des Sprungs vom letzten zurück zum ersten Eintrag in
`colorAnimList`) gleichermaßen, da beide vorher unter derselben
Bucket-Quantisierung litten.

## Text-Größe (viewport-relativ)

`S.textSize` (logische px, von `buildTextField()` verwendet) wird synchron
zu `S.textSizePct` (% der Viewport-Breite) gehalten — neu berechnet bei
Resize, Inhalts-/Gewichts-/Prozent-Änderung, damit der Text auf jedem
Viewport (Desktop wie Mobile, nach Rotation/Resize) automatisch die
Zielbreite trifft. Das Mobile-Preset nutzt dieselbe Logik mit reduzierter
Partikelanzahl; die Textbreite passt sich dadurch automatisch an jede
Bildschirmgröße an.

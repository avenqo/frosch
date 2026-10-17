# Frosch – Verdrahtung v0.7

Grundlage: `doc/spec.md` (Stand 14.10.2026, 17:01). Frosch besteht aus mehreren Sensorknoten im selben WLAN, jeder mit eigener ESPHome-Konfiguration.

| Knoten | Board | Firmware | MQTT-Präfix |
|---|---|---|---|
| Indoor: Wohnzimmer | ESP8266 NodeMCU V0.9 | `software/frosch-wohnzimmer.yaml` | `frosch/wohnzimmer/…` |
| Outdoor: Norden | ESP8266 NodeMCU V2 | `software/frosch-nord.yaml` | `frosch/nord/…` |

WLAN, MQTT und OTA stehen für alle Knoten gemeinsam in `software/common.yaml`.

**Schaltpläne (KiCad 7, öffnen auch mit KiCad 8/9):** `hardware/kicad/wohnzimmer/frosch-wohnzimmer.kicad_pro` und `hardware/kicad/nord/frosch-nord.kicad_pro`, gemeinsame Symbolbibliothek `hardware/kicad/frosch.kicad_sym`. Jeweils mit PDF-Export daneben.

**Änderungen gegenüber v0.6**
- Norden: BME280 (I2C, Temperatur/Feuchte/Druck) statt AM2302. D7 ist dadurch frei.

## Versorgung (beide Knoten)

Versorgt wird jeder Knoten von einem USB-Steckernetzteil (≥ 1 A), das immer innen steht; nach draußen geht nur das Kabel. Die 5 V werden **an einer Stelle** aufgeteilt (Stern), z. B. an einem USB-Buchsen-Breakout oder einer Schraubklemme:

```
USB-Netzteil 5 V ──► USB-Breakout / Klemme 5V ─┬── NodeMCU Vin
                                               ├── 5-V-Verbraucher (SPS30, HC-SR501, MQ-2)
                                               └── 100–470 µF Elko gegen GND
                          Klemme GND ──────────┴── sternförmig zu allen GND
```

- Die USB-Buchse des NodeMCU dient nur noch zum Flashen. Beim Flashen das Netzteil abziehen.
- **Bauteile mit 3,3 V** hängen am 3V3-Pin des NodeMCU: BMP280, BME280, ADS1115, LDR-Teiler und Regenplatine.

### Lehre aus der SPS30-Fehlersuche

Der SPS30 hat über I2C korrekt geantwortet, auch auf das Start-Kommando, aber Lüfter und Messung liefen nicht. Ursache war der Versorgungsweg über den Abgriff an der Diode und das Steckbrett.

- **Ohne Last** zeigte das Multimeter 4,6 V.
- **Beim Lüfteranlauf** brach die Spannung offenbar ein, und der SPS30 setzte sich zurück.
- **Merkmal im Log:** `Started measurements`, danach dauerhaft `Not ready`.
- **Prüfen:** Die Spannung **direkt am SPS30** (Pin 1 ↔ Pin 5) unter Last messen, Soll ≥ 4,5 V. Alternativ zum Test einen 100-Ω-Widerstand als Last anstelle des SPS30 anschließen.

## Indoor: Wohnzimmer (ESP8266 NodeMCU V0.9)

Der NodeMCU V0.9 hat dieselbe Pin-Zuordnung (D0–D8 ↔ GPIO) wie V2. Er ist aber breiter und belegt auf einem Standard-Steckbrett beide Lochreihen.

| Pin | GPIO | Funktion |
|---|---|---|
| D1 | 5 | I2C SCL → BMP280, ADS1115 |
| D2 | 4 | I2C SDA → BMP280, ADS1115 |
| 3V3 | – | BMP280, ADS1115 VDD, LDR |
| Vin | – | 5 V von der Klemme |
| GND | – | Stern-GND |

- **Frei:** D0 (16), D5 (14), D6 (12), D7 (13) und A0. Die Pins D3, D4 und D8 sind Boot-Strap-Pins und nur eingeschränkt nutzbar.

### I2C-Bus (D1/D2, 100 kHz)

| Gerät | Adresse | Hinweis |
|---|---|---|
| BMP280 | 0x76 | SDO → GND |
| ADS1115 | 0x48 | ADDR → GND |

Beide hängen parallel an SDA/SCL. Die Pull-ups bringen die Breakouts in der Regel mit, zusätzliche sind nicht nötig.

### BMP280
VCC → 3V3, GND → GND, SCL → D1, SDA → D2, SDO → GND.

### ADS1115
- **Versorgung:** VDD → **3V3**, GND → GND. SCL → D1, SDA → D2, ADDR → GND (Adresse 0x48).
- **Eingangsgrenze:** Die Eingänge dürfen **nie über VDD + 0,3 V** steigen, hier also 3,6 V. Deshalb braucht der MQ-2 einen Teiler.
- **Messbereich:** In der Firmware ist ±4,096 V eingestellt (`gain: 4.096`), passend für 0–3,3 V.
- **ALRT:** bleibt frei.

| ADS1115-Eingang | Signal |
|---|---|
| A0 | Fotowiderstand |
| A1 | MQ-2 AO (über Teiler) |
| A2, A3 | frei |

### Fotowiderstand an A0 (ADS1115)
```
3V3 ── LDR ──┬── A0 (ADS1115)
             │
           27 kΩ
             │
            GND
```
Die Firmware liefert die Spannung in Volt, 0 bis ca. 3,3 V, heller = größer. Eine Umrechnung in Lux gibt es nicht.

**Wahl des Festwiderstands:** Am größten ist der Spannungshub, wenn der Festwiderstand etwa dem LDR-Widerstand im interessanten Lichtbereich entspricht.
- **27 kΩ:** gut für Dämmerung und Dunkelheit.
- **10 kΩ (oder 6,8 kΩ):** mehr Hub zwischen hellem und schwach beleuchtetem Raum. Dafür liegen nachts alle Werte knapp über 0 V.

### MQ-2 an A1 (ADS1115)
```
MQ-2 AO ── 10 kΩ ──┬── A1 (ADS1115)
                   │
                 20 kΩ
                   │
                  GND
```
- **Versorgung:** VCC → **5 V** (Klemme), GND → GND. Die Heizung zieht dauerhaft ca. 150 mA.
- **Teiler:** 5 V am AO werden zu 3,33 V am ADS1115. Die Firmware rechnet mit Faktor 1,5 auf die Spannung am Modul zurück.
- **Messwert:** relativ, kein ppm. Die Schwelle für „Gas“ legst du selbst anhand der Werte fest.
- **DO:** bleibt frei.
- **Einbrennen:** Neue Sensoren 24–48 h einbrennen lassen. Nach jedem Einschalten einige Minuten Aufwärmzeit abwarten.
- **Abstand halten:** Die Heizung erwärmt die Umgebung. Den MQ-2 nicht direkt neben den BMP280 setzen.

## Outdoor: Norden (ESP8266 NodeMCU V2)

| Pin | GPIO | Funktion |
|---|---|---|
| D1 | 5 | I2C SCL → SPS30, BME280 |
| D2 | 4 | I2C SDA → SPS30, BME280 |
| D5 | 14 | HC-SR501 OUT |
| D6 | 12 | Regen DO |
| A0 | ADC | Helligkeit (LDR) |
| 3V3 | – | BME280, LDR, Regenplatine |
| Vin | – | 5 V von der Klemme |
| GND | – | Stern-GND |

**Frei:** D7 (13) und D0 (16). Die Pins D3, D4 und D8 sind Boot-Strap-Pins.

### I2C-Bus (D1/D2, 100 kHz)

| Gerät | Adresse | Versorgung |
|---|---|---|
| SPS30 | 0x69 | 5 V von der Klemme |
| BME280 | 0x76 (SDO → GND) | 3V3 |

### SPS30 (I2C, 0x69)
**Stecker JST ZHR-5:** 1 VDD → **5 V von der Klemme** · 2 SDA → D2 · 3 SCL → D1 · 4 SEL → GND · 5 GND → Stern-GND

- **Pull-ups:** Das BME280-Breakout bringt sie in der Regel mit (10 kΩ nach 3,3 V), zusätzliche sind dann nicht nötig. Ohne Pull-ups auf dem Breakout je 4,7 kΩ von SDA und SCL nach **3V3** setzen, nie nach 5 V.
- **Pegel:** I2C mit 3,3 V funktioniert, das hat der Kreuztest bestätigt.
- **Lange Zuleitung:** Das Netzteil steht innen, das Kabel geht nach draußen. Spannung am SPS30 unter Last messen, Sollwert ≥ 4,5 V. Bei langen oder dünnen Kabeln gibt es zwei Abhilfen: ein Kabel mit größerem Querschnitt oder ein Netzteil mit 5,1–5,25 V.

### BME280 (I2C, 0x76)
- **Anschluss:** VCC → 3V3, GND → GND, SCL → D1, SDA → D2, SDO → GND (Adresse 0x76). Bei 6-poligen Breakouts zusätzlich CSB → 3V3.
- **Montage:** im Strahlungsschutz (Lamellengehäuse), belüftet, ohne direkte Sonne und ohne Regen. Nicht im geschlossenen Gehäuse, nicht neben ESP oder SPS30-Abluft.
- **Nicht beschichten oder versiegeln:** Die Öffnung im Chip muss frei bleiben. Schutz durch Gitter oder PTFE-Membran ist in Ordnung.
- **Kurze I2C-Leitung:** ESP und Sensor am besten im selben Strahlungsschutz, sonst höchstens ca. 50 cm Kabel.
- **Grenzen:** Bei langer Kondensation (Nebel, über 90 % Feuchte) driftet die Feuchte-Messung nach oben und erholt sich langsam. Für höhere Ansprüche an die Feuchte wäre ein SHT40/SHT45 besser.

### Helligkeit (A0)
```
3V3 ── LDR ──┬── A0
             │
           27 kΩ
             │
            GND
```
Die Firmware liefert den Rohwert 0–1023, heller = größer. Das NodeMCU hat an A0 einen eigenen Teiler (220k/100k) gegen GND, der parallel zu den 27 kΩ liegt.

### HC-SR501 (PIR)
- **Anschluss:** VCC an 5 V (Klemme), GND an GND, OUT an D5. Der Ausgang liefert 3,3 V.
- **Jumper:** auf H (retriggerbar).
- **Zeit-Poti:** 5–30 s.
- **Gehäuse:** PETG ist nicht IR-durchlässig. Die Fresnel-Kuppel muss frei herausschauen.

### Regensensor
- **Anschluss:** VCC an 3V3, GND an GND, DO an D6, LOW = nass.
- **Schwelle:** am Poti des Moduls einstellen.
- **Korrosion:** Die Platine korrodiert unter Dauerspannung. Bei Bedarf VCC aus D0 schalten (Firmware-Änderung).

### Erweiterung
- **I2C-Stecker:** Einen 4-poligen Stecker (3V3, GND, SDA, SCL) nach außen führen.
- **Windsensor:** Ein Windsensor mit Reed-Impulsen kann an D7 (GPIO13) und wird in ESPHome mit `pulse_counter` ausgewertet. Eine Windfahne (analog) bräuchte einen ADS1115 am I2C-Bus, weil A0 belegt ist.

## Stückliste

| Knoten | Teile |
|---|---|
| Wohnzimmer | NodeMCU V0.9, BMP280-Breakout, ADS1115-Modul, LDR (GL5528) + 27 kΩ, MQ-2-Modul, 10 kΩ + 20 kΩ (Teiler MQ-2), USB-Breakout/Klemme, Elko 100–470 µF, USB-Netzteil ≥ 1 A |
| Norden | NodeMCU V2, SPS30 + JST-ZHR-5-Kabel, BME280-Breakout, ggf. 2× 4,7 kΩ (I2C-Pull-ups), LDR + 27 kΩ, HC-SR501, Regenplatine + LM393-Modul, USB-Breakout/Klemme, Elko 100–470 µF, USB-Netzteil ≥ 1 A, Außenkabel |

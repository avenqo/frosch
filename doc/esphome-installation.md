# ESPHome: Installation und Nutzung

Gilt für Ubuntu/Debian mit System-Python 3.12. Dort ist `pip install` systemweit gesperrt (PEP 668), deshalb läuft alles über **pipx**. pipx legt für jedes Programm eine eigene Umgebung an und stellt die Befehle in `~/.local/bin` bereit.

## a) Installation (einmalig)

```bash
# 1. pipx installieren und in den PATH eintragen
sudo apt install pipx
pipx ensurepath
# → danach ein neues Terminal öffnen

# 2. ESPHome (Kommandozeile)
pipx install esphome

# 3. ESPHome Device Builder (Weboberfläche), mit eigenem ESPHome
pipx install 'esphome-device-builder[esphome]'

# 4. Zugriff auf USB-Ports (/dev/ttyUSB*) erlauben
sudo usermod -aG dialout $USER
# → danach einmal ab- und wieder anmelden

# 5. Zugangsdaten anlegen (secrets.yaml ist in .gitignore)
cd ~/git/frosch/software
cp secrets.example.yaml secrets.yaml
nano secrets.yaml
```

Prüfen:
```bash
esphome version
esphome-device-builder --version
```

Updates für beide:
```bash
pipx upgrade-all
```

## b) Nutzung in einem neuen Terminal

Es muss nichts aktiviert werden. Die Befehle liegen nach `pipx ensurepath` dauerhaft im PATH. Du wechselst nur in den Projektordner:

```bash
cd ~/git/frosch/software
```

### Kommandozeile

| Zweck | Befehl |
|---|---|
| Konfiguration prüfen | `esphome config frosch-wohnzimmer.yaml` |
| Nur kompilieren | `esphome compile frosch-wohnzimmer.yaml` |
| Kompilieren + flashen (fragt nach USB oder OTA) | `esphome run frosch-wohnzimmer.yaml` |
| Flashen per USB | `esphome run frosch-wohnzimmer.yaml --device /dev/ttyUSB0` |
| Flashen per WLAN (OTA) | `esphome run frosch-wohnzimmer.yaml --device frosch-wohnzimmer.local` |
| Log per USB (inkl. Startvorgang) | `esphome logs frosch-wohnzimmer.yaml --device /dev/ttyUSB0` |
| Log drahtlos (über MQTT) | `esphome logs frosch-wohnzimmer.yaml` |

Für den Nord-Knoten entsprechend `frosch-nord.yaml`. Den USB-Port findest du mit `ls /dev/ttyUSB* /dev/ttyACM*`.

### Weboberfläche (Device Builder)

```bash
esphome-device-builder --host 127.0.0.1 ~/git/frosch/software
```

Dann im Browser `http://localhost:6052` öffnen. Beenden mit `Strg+C`.

- **Zugriff aus dem Netz:** Ohne `--host 127.0.0.1` ist die Oberfläche von allen Rechnern im Netz erreichbar.

### Wenn ein Befehl nicht gefunden wird

```bash
pipx ensurepath        # trägt ~/.local/bin in den PATH ein
exec $SHELL            # Terminal neu laden
```

Der Device Builder meldet „needs the 'esphome' package“? Dann wurde er ohne `[esphome]` installiert:
```bash
pipx install --force 'esphome-device-builder[esphome]'
```

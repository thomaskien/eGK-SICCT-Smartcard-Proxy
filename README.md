# eGK-SICCT-Smartcard-Proxy
<img src="https://github.com/thomaskien/eGK-SICCT-Smartcard-Proxy/blob/main/IMG_3099.jpeg?raw=true" alt="drawing" width="800" />

# SICCT-Proxy für T2med mit einfachem USB-Kartenleser

Dieses Projekt stellt einen lokalen **SICCT-kompatiblen Proxy** bereit, damit **T2med** einen einfachen **kontaktbehafteten USB-Smartcard-Reader** wie den **Alcor Link AK9563** als Kartenleser verwenden kann.

Der Proxy nimmt SICCT-Verbindungen von T2med entgegen und reicht die eigentlichen Karten-APDUs an den lokal angeschlossenen Leser weiter.

Aktueller Fokus:
- läuft unter Linux oder Windows
- **eGK lesen**
- **T2med über SICCT ansprechen** - vielleicht gehen auch andere PVS
- einfacher USB-Leser statt klassischem LAN-Kartenterminal
- Reader für 12,99 Euro bei Amazon "CSL Smartcard USB"
- gibt es für USB und USB-C

---
## T2med
Kartenleser einrichten:
- URL: http://IP.DES.RECHNERS:4742
- benutzer: beliebig
- passwort: beliebig
- Achtung: localhost geht nicht, da der SERVER die verbindung zum kartenleser aufbaut, nicht der client!
- wenn mal was nicht geht: unter kartenlesegeräte den "play" knopf drücken, dann wird die session repariert


## Funktionsumfang

Der Proxy unterstützt aktuell unter anderem:

- `INIT CT SESSION`
- `CLOSE CT SESSION`
- `REQUEST ICC`
- `RESET CT / ICC`
- `GET STATUS`
- `OUTPUT`
- `EJECT ICC`
- Weiterreichen normaler Karten-APDUs an die eGK
- Slot-Ereignis **„Karte entfernt (85)”** nach `EJECT ICC`

---

## Voraussetzungen

### Hardware

- Linux-Rechner
- oder Windows-Rechner
- kontaktbehafteter Smartcard-Reader
- getesteter Reader:
  - **Alcor Link AK9563**
- eGK / Gesundheitskarte

### Software

- Python 3
- `pcscd`
- `pcsc-tools`
- `libccid`
- `pyscard`

---

## Installation der benötigten Pakete

### Ubuntu / Debian

```bash
deactivate 2>/dev/null || true

sudo apt update
sudo apt install -y \
  python3-dev \
  python3.12-dev \
  build-essential \
  swig \
  pkg-config \
  libpcsclite-dev \
  pcscd \
  pcsc-tools \
  libccid

python3 -m venv ~/venvs/sicct
source ~/venvs/sicct/bin/activate

python -m pip install --upgrade pip setuptools wheel
python -m pip install pyscard
````

### PC/SC-Dienst aktivieren

```bash
sudo systemctl enable --now pcscd
```


---

## Reader-Funktion testen

Zuerst prüfen, ob der Leser unter Linux korrekt erkannt wird:

```bash
pcsc_scan
```

Typischer erfolgreicher Output:

* Reader wird erkannt
* Karte wird erkannt
* ATR wird angezeigt

Beispiel:

```text
Reader 1: Alcor Link AK9563 01 00
Card state: Card inserted
ATR: 3B D3 96 FF 81 B1 FE 45 1F 07 80 81 05 2D
```

---

## Projektdatei

Speichere den Python-Proxy z. B. als:

```text
sicct12.py
```

oder einen späteren Stand mit anderem Dateinamen.

---

## Konfiguration im Script

Im Script müssen diese Werte geprüft bzw. angepasst werden:

```python
READER_INDEX = 1
EXPECTED_READER_SUBSTR = "Alcor Link AK9563"
```

### Bedeutung

* `READER_INDEX`
  Index des verwendeten Readers in `smartcard.System.readers()`

* `EXPECTED_READER_SUBSTR`
  Sicherheitsprüfung, damit nicht versehentlich der falsche Reader verwendet wird

---

## Verfügbare Reader anzeigen

```bash
python3 - <<'PY'
from smartcard.System import readers
for i, r in enumerate(readers()):
    print(i, r)
PY
```

---

## Proxy starten

```bash
source ~/venvs/sicct/bin/activate
python3 sicct12.py
```

Beispielausgabe:

```text
Verfügbare Reader:
  [0] ACS ACR122U PICC Interface 00 00
  [1] Alcor Link AK9563 01 00
Gewählter Reader: Alcor Link AK9563 01 00
lausche auf 0.0.0.0:4742
```

---

## T2med konfigurieren

In T2med den Leser als **LAN-Kartenleser** eintragen.

### URL

```text
http://<IP-DES-PROXY-SERVERS>:4742
```

Beispiel:

```text
http://10.0.83.5:4742
```

### Zugangsdaten

Aktuell erwartet der Proxy typischerweise:

* Benutzername: `test`
* Passwort: `test`

Diese Werte werden von T2med beim `INIT CT SESSION` mitgesendet.

---

## Netzwerk

Falls T2med auf einem anderen Rechner läuft, muss Port **4742/TCP** erreichbar sein.

Test auf dem Proxy-Rechner:

```bash
ss -ltnp | grep 4742
```

---

## Typischer Ablauf im Log

Ein erfolgreicher Ablauf sieht etwa so aus:

1. `INIT CT SESSION`
2. `REQUEST ICC`
3. `RESET CT / ICC`
4. Karten-APDUs an die eGK
5. Auslesen der Daten
6. `EJECT ICC`
7. Slot-Ereignis **Karte entfernt (85)**

---

## Wichtige Hinweise

### 1. Reader-Zustand

Der Proxy arbeitet aktuell mit einem **globalen Reader-/Kartenzustand**, weil physisch nur ein Reader vorhanden ist.

### 2. Kein klassisches TI-Kartenterminal

Dieses Projekt ist **kein zugelassenes eHealth-Kartenterminal** und **kein TI-Konnektor-Ersatz**.
Es ist ein technischer Proxy, um mit T2med und frei lesbaren Kartendaten zu experimentieren bzw. lokale Arbeitsabläufe zu ermöglichen.

### 3. Session-Verhalten

Die SICCT-Session wird durch T2med aufgebaut.
Der Proxy hält die Verbindung offen und reagiert auf SICCT-Kommandos von T2med.

### 4. EJECT-Verhalten

Nach `EJECT ICC` wird aktuell die Karte lokal deaktiviert und zusätzlich ein Slot-Ereignis **„Karte entfernt (85)”** an T2med gesendet.

---

## Troubleshooting

### Reader wird nicht gefunden

Prüfen:

```bash
pcsc_scan
```

Wenn dort kein Reader erscheint:

* USB-Verbindung prüfen
* `pcscd` prüfen
* `libccid` installiert?
* Reader-Index im Script korrekt?

---

### Falscher Reader wird gewählt

Reader-Liste anzeigen:

```bash
python3 - <<'PY'
from smartcard.System import readers
for i, r in enumerate(readers()):
    print(i, r)
PY
```

Dann `READER_INDEX` und `EXPECTED_READER_SUBSTR` im Script anpassen.

---

### T2med meldet Kommunikationsfehler

Prüfen:

* stimmt die URL?
* ist Port 4742 erreichbar?
* läuft der Proxy?
* kommt beim Start von T2med überhaupt ein `INIT CT SESSION` im Log an?
* ist der richtige Reader ausgewählt?

---

### Python stürzt wegen fehlendem Reader ab

Der aktuelle Stand des Scripts sollte den Fall fehlender Reader robuster behandeln.
Trotzdem gilt: Wenn gar kein passender Reader vorhanden ist, kann natürlich keine Karte gelesen werden.

---

### Karte wird unter Linux gelesen, aber nicht in T2med

Dann ist meist einer dieser Punkte die Ursache:

* falscher Reader
* Session-/SICCT-Zustand
* T2med erwartet ein bestimmtes Terminalverhalten
* LAN-Zugriff/Firewall falsch
* T2med hält eine alte Verbindung offen

---

## Praktische Testschritte

### 1. Linux-Test

```bash
pcsc_scan
```

### 2. Proxy starten

```bash
source ~/venvs/sicct/bin/activate
python3 sicct12.py
```

### 3. T2med verbinden

LAN-Kartenleser in T2med auf den Proxy zeigen lassen.

### 4. Karte einlesen

Karte einstecken und T2med-Protokoll / Proxy-Log beobachten.

---

## Optional: Start über systemd

Beispiel-Service-Datei:

```ini
[Unit]
Description=SICCT Proxy for T2med
After=network.target pcscd.service
Requires=pcscd.service

[Service]
Type=simple
User=thomas
WorkingDirectory=/home/thomas/sicct-probe
ExecStart=/home/thomas/venvs/sicct/bin/python /home/thomas/sicct-probe/sicct12.py
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Speichern als:

```text
/etc/systemd/system/sicct-proxy.service
```

Dann:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sicct-proxy.service
sudo systemctl status sicct-proxy.service
```

---

## Logausgabe per systemd ansehen

```bash
journalctl -u sicct-proxy.service -f
```

---

## Projektstatus

Aktuell ist das Projekt ein funktionaler SICCT-Proxy-Ansatz für:

* einfachen USB-Reader
* eGK-Lesen
* T2med-Anbindung per LAN-SICCT

Das Verhalten einzelner Terminal-Kommandos kann je nach T2med-Version oder Kartenleser-Testlogik noch weiter verfeinert werden.

---

## Lizenz / Nutzungshinweis

Nutzung auf eigenes Risiko.
Keine Gewähr für TI-Konformität, Zulassungsfähigkeit oder Abrechnungsfähigkeit.

```

# DepuTask – Programme zum Herunterladen

**DepuTask – Deputatsplanung für Schulen.** Dieses Verzeichnis enthält ausschließlich die fertigen
Programme (kein Quelltext). Es wird bei jeder neuen Version vollständig ersetzt; die installierten
Programme und Container finden neue Versionen hier selbst (`version.json`).

DepuTask ist proprietäre Software. Die Nutzung richtet sich nach dem [Lizenzvertrag](LICENSE);
ohne gültige Lizenzdatei läuft DepuTask im Lesemodus. Lizenzhinweise der enthaltenen
Fremdkomponenten: [THIRD_PARTY_NOTICES.txt](THIRD_PARTY_NOTICES.txt). Änderungen je Version:
[CHANGELOG.md](CHANGELOG.md).

## Desktop-App

| System | Datei |
|---|---|
| Windows (x64) | [`builds/deputask-windows-amd64.exe`](builds/deputask-windows-amd64.exe) |
| Windows (ARM) | [`builds/deputask-windows-arm64.exe`](builds/deputask-windows-arm64.exe) |
| Linux (x64, benötigt WebKitGTK 4.1) | [`builds/deputask-linux-amd64`](builds/deputask-linux-amd64) |

Herunterladen und starten, eine Installation ist nicht nötig. Die App prüft beim Start, ob eine neue
Version vorliegt, und aktualisiert sich auf Wunsch selbst (**Über DepuTask → Aktualisierung**).

## Server (Schulnetz)

### Als Container

```bash
mkdir -p data
DEPUTASK_UID=$(id -u) DEPUTASK_GID=$(id -g) docker compose up -d     # compose.yaml aus diesem Verzeichnis
# oder ohne Compose
docker run -d --name deputask --user "$(id -u):$(id -g)" -p 8080:8080 -v "$PWD/data:/data" ghcr.io/tna76874/deputask:latest
```

Danach läuft DepuTask auf http://localhost:8080. Alle Daten liegen im lokalen Verzeichnis `./data`,
das direkt in den Container eingebunden wird (Bind-Mount, kein Docker-Volume): Datenbank,
Sicherungen, eingespielte Aktualisierungen, optional `license.lic` und `config.json`. Der Container
läuft mit der Benutzer-ID, der das Verzeichnis gehört – deshalb `--user` bzw. `DEPUTASK_UID`.

Laufende Container prüfen regelmäßig, ob eine neue Version
vorliegt, und zeigen dann einen Hinweis in der **Übersicht**. Ein Klick lädt die neue Version,
prüft ihre SHA-256-Prüfsumme, legt sie im Datenverzeichnis ab (`./data/update`) und startet den Server neu –
ohne neues Image. Die Aktualisierung bleibt auch nach einem Neustart oder Neuanlegen des Containers
aktiv, solange das Datenverzeichnis besteht; ein neueres Image hat Vorrang. Startet eine eingespielte Version
dreimal nicht, läuft wieder die Version aus dem Image.

Abschalten: `./data/config.json` mit `{"update": {"enabled": false}}`. Das Dateisystem von `./data`
darf nicht mit `noexec` eingehängt sein.

### Als Programm

`builds/deputask-server-<system>-<architektur>` für Linux, Windows und macOS:

```bash
./deputask-server-linux-amd64 -addr 127.0.0.1:8080 -db deputask.db            # nur melden, dass es Neues gibt
./deputask-server-linux-amd64 -addr 127.0.0.1:8080 -db deputask.db -self-update   # darf sich selbst ersetzen
```

## Prüfsummen

`version.json` nennt zu jeder Datei Größe und SHA-256. Von Hand prüfen:

```bash
sha256sum -c SHA256SUMS
```

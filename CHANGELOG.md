# Changelog

Alle nennenswerten Änderungen an DepuTask. Format nach [Keep a Changelog](https://keepachangelog.com/de/1.1.0/),
Versionierung nach [Semantic Versioning](https://semver.org/lang/de/) – Details in [`docs/VERSIONING.md`](docs/VERSIONING.md).
Jede Version nennt die **Schema-Version** der Datenbank, die sie mitbringt.

## [Unreleased]

## [0.10.0] – 2026-11-14

Schema-Version: **3** · Lizenzformat: **1**

### Neu
- Lizenzverwaltung: Rechnungen als PDF je Lizenz (fortlaufende Nummern, §-14-Pflichtangaben, Kleinunternehmer-Option),
  Zahlungen, Mahnungen, Stornorechnungen, Protokoll, CSV-Export; Schema der Lizenzverwaltung 2
- Inventar der Fremdkomponenten mit Lizenzprüfung (`make third-party`), Lizenztexte eingebettet unter „Über DepuTask“
- Eigener Lizenzvertrag (EULA, deutsches Recht) als `LICENSE`, unter „Über DepuTask“ einsehbar; Copyright-Hinweise in
  allen Quelltexten und Programmen aus der zentralen Konfiguration (`make copyright`)
- Anwendungsweite Konfiguration (`backend/config/config.json`, Überschreiben von `update`/`auth` je Installation)
- Selbstaktualisierung: Desktop-App ersetzt sich nach Prüfsummen-Check selbst; Container prüfen regelmäßig, zeigen
  einen Hinweis in der Übersicht und laden neue Versionen in das eingebundene Datenverzeichnis (übersteht Neustarts)
- `make release`: Programme, `version.json` mit SHA-256 und Docker-Workflow als Commit ohne Historie ins Release-Repo
- Optionale Anmeldung über OpenID Connect mit Rollen (Administration, Planung, Sekretariat, Lesen) und Benutzerverwaltung
- Lizenzverwaltung: Lizenztyp „kostenlose Überlassung“ mit Beleg ohne Steuer- und Bankangaben, Vereinbarungen aus
  Vorlagen (kostenlos/Standard), Schema der Lizenzverwaltung 3

### Geändert
- Docker-Laufzeit-Images auf `scratch` umgestellt (nur das Binary, keine Betriebssystempakete)
- Docker: Daten nur im eingebundenen lokalen Verzeichnis `/data` (kein `VOLUME`, Start mit `--user`)
- Planungsdatenbank Schema 3: Benutzer und Sitzungen (nur mit Anmeldung genutzt)

## [0.9.0] – 2026-11-08

Schema-Version: **2** · Lizenzformat: **1** · erste Version mit Datenbank-Migrationen

### Neu
- Deputatsplanung: Unterrichtsmatrix mit Infofenster, Koppeln, Halbjahre, Bilanz WStd Soll/Ist/Diff
- Klassenbildung, Bedarfsplanung, automatische Lehrauftragsverteilung
- CSV-/XLSX-Import (ASV-BW) und -Export, Wiederherstellungspunkte
- Desktop-App (Wails), Webserver, Docker-Builds
- Offline-Lizenzierung (Ed25519), ohne gültige Lizenz nur Lesen
- Versionierte Datenbank-Migrationen (goose) mit automatischer Sicherung vor jedem Update

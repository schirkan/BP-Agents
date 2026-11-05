---
name: bp-log-viewer-cli
description: "Verwende diesen Skill für BP Log Viewer CLI: Blue-Prism-Sessions, Logs und Exceptions analysieren, abgebrochene Prozesse untersuchen, Ressourcen, Work Queues und Audit-Ereignisse abfragen. Enthält die vollständige Parameterreferenz, PowerShell-Beispiele, Filterwirkung, JSONL-Ausgabe, Konfiguration und Exit-Codes."
argument-hint: "Gewünschte Abfrage oder Fehleranalyse, möglichst mit Session-Nummer, Prozess, Ressource und Zeitraum."
---

# BP Log Viewer CLI

## Zweck

Das CLI liest Daten einer Blue-Prism-7.5.x-Datenbank (SQL Server oder LocalDB) und liefert strukturierte JSONL-Ergebnisse. Blue Prism ist ein RPA-Tool zur Automatisierung von Geschäftsprozessen; eine Session ist eine einzelne Prozessausführung. Dieses CLI analysiert bestehende Daten, startet oder stoppt aber keine Blue-Prism-Prozesse.

## Referenz

Lies vor der Auswahl von Parametern die [vollständige CLI-Referenz](./references/cli-reference.md). Sie dokumentiert alle implementierten Befehle und Optionen, ihre tatsächliche Wirkung und bekannte Einschränkungen. Nicht jede global akzeptierte Option wirkt auf jeden Befehl.

## Lokale CLI und Voraussetzungen

- Verwende `bin\bp-log-viewer.exe` relativ zum Ordner dieser Skill-Datei. Die lokale Verbindungskonfiguration liegt daneben unter `bin\appsettings.json`; benötigte DLLs gehören ebenfalls in diesen Ordner.
- Das Binary stammt aus dem Repository [**bp-log-viewer-cli**](https://github.com/schirkan/bp-log-viewer-cli) (Projekt `src/BpLogViewer.Cli/BpLogViewer.Cli.csproj`; framework-dependent single file, benötigt .NET 8 Runtime x64).
- **Fertige Release-Builds gibt es.** Unter [Releases](https://github.com/schirkan/bp-log-viewer-cli/releases) liegt je Version ein ZIP `bp-log-viewer-win-x64-<version>.zip` plus `<...>.zip.sha256`. Download-Pfad: `https://github.com/schirkan/bp-log-viewer-cli/releases/download/<version>/bp-log-viewer-win-x64-<version>.zip`. **Nimm immer das neueste Release** - ältere Releases können ein anderes Ausgabeformat liefern als das hier Beschriebene.
- **Lade das Binary nicht selbst herunter und baue es nicht selbst.** Fehlt es, weise den Nutzer an, das Paket aus den [Releases](https://github.com/schirkan/bp-log-viewer-cli/releases) herunterzuladen, die Prüfsumme gegen die `.sha256`-Datei abzugleichen, es zu entpacken und `bp-log-viewer.exe` **manuell** unter `bin\bp-log-viewer.exe` im Skill-Ordner abzulegen. Die Verbindungskonfiguration legt er daneben unter `bin\appsettings.json` ab; das Paket enthält eine `appsettings.json` auf Basis der Vorlage, die er mit den echten Verbindungsdaten füllt. Nenne Version, Download-URL, Zielpfad und Runtime-Voraussetzung und warte auf die Bestätigung, dass die Dateien liegen.
- Löse den Binary-Pfad vor dem Aufruf zu einem absoluten Windows-Pfad auf. Er ist relativ zum Skill-Ordner, nicht zum beliebigen aktuellen Arbeitsverzeichnis. Verlasse dich nicht auf die PATH-Suche.
- Das CLI sucht `appsettings.json` zuerst im Binary-Verzeichnis (nach einem nur intern verwendeten expliziten Basisverzeichnis), danach im Arbeitsverzeichnis und dessen Parent. Vorhandene ENV-Verbindungswerte können die Dateikonfiguration übersteuern; siehe [Konfiguration](./references/cli-reference.md#8-konfiguration-und-zeitangaben).
- Prüfe nur die Existenz der lokalen Konfiguration; ihren Inhalt mit echten Verbindungsdaten nicht lesen, anzeigen oder kopieren. Verwende die eingerichtete Verbindung ohne Zugangsdaten in Argumenten.
- Der gesamte lokale `bin`-Ordner ist über die Root-`.gitignore` ausgeschlossen. Binaries und Verbindungskonfiguration nicht committen oder hochladen; die Ignore-Regel entfernt keine bereits versionierten Dateien.
- Fehlen Binary, Konfiguration oder benötigte Runtime, melde die fehlende Voraussetzung und die oben beschriebene manuelle Ablage. Nicht ungefragt bauen, installieren, herunterladen oder die Konfiguration ändern.

## Vorgehen

1. Kläre Ziel, Kennung und Zeitraum. Frage nach, wenn der betroffene Prozess oder die Session nicht eindeutig ist.
2. Prüfe die lokale CLI unter `bin\bp-log-viewer.exe` und die danebenliegende `appsettings.json`. Verwende den aufgelösten absoluten Binary-Pfad und in PowerShell den Aufrufoperator `&`. Ein anderer Installationsort muss ausdrücklich benannt sein.
3. Prüfe das Binary einmalig mit `version` - das bestätigt, dass es startet und die .NET-8-Runtime x64 vorhanden ist. Verwende `self-test` nur zur Diagnose von Konfiguration oder Verbindung. Beachte: `self-test` schreibt und löscht eine temporäre Testdatei und kann ein `temp`-Verzeichnis hinterlassen.
4. Wähle anhand der Referenz den engsten geeigneten Befehl. Verwende explizite Zeitangaben mit `Z` oder Offset und zunächst ein kleines `--max-rows`.
5. Prüfe stdout, stderr und den unmittelbar nach dem Aufruf erfassten Exit-Code. Parse JSONL zeilenweise, nicht als JSON-Array. Ausnahme: `session log` liefert kompakte Arrays - die erste Zeile ist der Spaltenkopf, danach folgt je Log-Eintrag ein Array mit 8 Werten in derselben Spaltenreihenfolge. Warnungen und leere Ausgabe sind keine Belege für Fehlerfreiheit.
6. Verfeinere Filter oder erhöhe das Limit, wenn entscheidende Daten fehlen. Bei Log-Trunkierung fehlen die späteren Einträge, möglicherweise gerade der Abbruch. Es gibt keine serverseitige Log-Paginierung oder Zeitfilterung.

## Fehleranalyse eines abgebrochenen Prozesses

Die Beispiele verwenden illustrative Namen, Session-Nummern und Zeitangaben. Ersetze sie durch Werte aus der Anfrage bzw. aus den Ergebnissen. Die folgende Pfadauflösung setzt das Repository-Root als aktuelles Arbeitsverzeichnis voraus; bei anderem Arbeitsverzeichnis den absoluten Skill-Ordner verwenden.

```powershell
$cli = (Resolve-Path -LiteralPath '.github\skills\bp-log-viewer-cli\bin\bp-log-viewer.exe' -ErrorAction Stop).Path
& $cli session list --process 'Rechnungsverarbeitung' --since '2026-10-15T00:00:00Z' --until '2026-10-16T00:00:00Z' --max-rows 50
$exitCode = $LASTEXITCODE
```

- Suche nicht ausschließlich mit `--has-exception`: Ein manuell gestoppter oder anderweitig abgebrochener Lauf muss keine Session-Exception haben.
- Übernimm `session_number` aus dem Ergebnis; Statusnamen werden aus der Datenbank geladen, nicht als feste Codes angenommen.

```powershell
& $cli session get 4711
$exitCode = $LASTEXITCODE
& $cli session log 4711 --level error --max-rows 200
$exitCode = $LASTEXITCODE
& $cli session log 4711 --max-rows 2000
$exitCode = $LASTEXITCODE
```

- Vergleiche Session-Exception, `last_stage` und Log-Einträge. Die Fehlerfilterung umfasst Resulttypen 2 und 64, aber nicht Typ 4 (`Exception`); lies deshalb auch ungefilterte Logs.
- Rekonstruiere die ursprüngliche Exception und mögliche Folgefehler. Ein Fehler im Log allein beweist nicht die Ursache der Terminierung.
- Verwende `--stage '<Stage-Name>' --full-result` gezielt, wenn Vorschauen nicht ausreichen. Ohne Ergebnisdatei sind maximal 100000 Zeilen möglich; auch eine Datei hebt diese Obergrenze nicht auf.
- Korrelierte Queue-Items und Audit-Ereignisse können Hinweise auf Datenfehler oder Bedienhandlungen liefern. Korrelation ist kein Kausalitätsbeweis.
- Für wiederkehrende Fehler: `session errors --aggregate` mit explizitem Zeitraum; danach einzelne `sample_session_numbers` untersuchen. Die Aggregation wird nicht durch `--max-rows` begrenzt.

## Quelle / Source

Die beschriebene CLI lebt im Repository [**bp-log-viewer-cli**](https://github.com/schirkan/bp-log-viewer-cli). Ergänzungen, Korrekturen und Issues bitte dort einreichen. Das Binary für die lokale Ablage unter `bin\` stammt aus den [Releases](https://github.com/schirkan/bp-log-viewer-cli/releases) desselben Repositorys; der Agent lädt es nicht selbst herunter und baut es nicht selbst.

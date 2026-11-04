---
name: bp-analyzer-cli
description: "Verwende diesen Skill, um einen Blue-Prism-Prozess als XML zu exportieren und in lesbaren VB.NET-Code samt JSON-Metadaten zu konvertieren, damit der Prozess-Ablauf analysierbar wird. Deckt export, codegen und das optionale analyze für SDD-Dokumentation ab."
argument-hint: "Prozessname, Zielumgebung und gewünschter Schritt: export, codegen oder (optional) analyze."
---

# BP Analyzer CLI

## Zweck

Die Blue-Prism-XML-Struktur ist für KI schwer lesbar: Prozesslogik, Datenitems, Stagetypen und Exception-Handling liegen tief verschachtelt in Attributen und Referenzen. Das BP-Analyzer-CLI löst das in zwei Schritten:

1. **export** — der Prozess wird über die Blue-Prism-CLI (`AutomateC.exe`) als XML exportiert, inklusive aller Abhängigkeiten.
2. **codegen** — aus dem XML entstehen lesbare **VB.NET-Klassen** mit GoTo-basierter Ablaufsteuerung sowie **JSON-Metadaten** (`classes.json`, `dependencies.json`, `exceptions.json`).

Optional und nur auf ausdrücklichen Wunsch:
3. **analyze** — aus den JSON-Metadaten entsteht ein **SDD-Dokument** (Solution Design Document) in Markdown.

Der erzeugte Code dient der **statischen Analyse des Ablaufs**: sichtbar werden, in welcher Reihenfolge Stages laufen, wie Daten zwischen ihnen fließen, wo Exception-Handling greift und welche Prozesse einander aufrufen. Er bildet das Laufzeitverhalten **nicht** ab.

## Input

| Feld | Pflicht | Bedeutung |
|---|---|---|
| `process_name` | ja | Exakter Name des Blue-Prism-Prozesses in der Zielumgebung. Der Export ist namensabhängig; bei falscher Schreibweise schlägt er fehl. |
| `target` | ja | Zielumgebung, gegen die `AutomateC.exe` arbeitet. |
| `mode` | ja | `export`, `codegen` oder `analyze` (optional). |
| `exception_exclusions` | nein | Nur für `analyze`: kommagetrennte Präfixe von Exception-Quellen, die im SDD ausgelassen werden. Standard: `ERGO_,Utility_`. |
| `credentials` | bei Bedarf | Anmeldedaten für die Zielumgebung. SSO bevorzugen, nie als Argument dokumentieren. |

## Output

| Modus | Ergebnis |
|---|---|
| `export` | XML-Dateien im Zielverzeichnis: `BPA Process - <name>.xml` sowie je Abhängigkeit eine weitere XML-Datei. Zusammenfassung über Succeeded/Failed mit Dateiliste. |
| `codegen` | Pro XML eine `.vb`-Datei (VB.NET, net48) mit GoTo-Ablaufsteuerung, dazu `classes.json`, `dependencies.json`, `exceptions.json` und `BluePrism_Generated.vbproj`. |
| `analyze` | SDD-Markdown pro Prozess, aufgebaut aus Klassen-, Abhängigkeits- und Exception-Liste. |

Für die Ablaufanalyse relevant sind:

| Artefakt | Was daran ablesbar ist |
|---|---|
| `<Prozess>.vb` | Stage-Reihenfolge über GoTo-Labels, Zuweisungen an Datenitems, `On Error GoTo`-Verzweigungen zu Recover- und Resume-Stages |
| `dependencies.json` | beteiligte Prozesse, Business Objects und SubSheets |
| `classes.json` | erzeugte Klassen und ihre Herkunft |
| `exceptions.json` | aufgefangene und weitergereichte Exceptions |

## Voraussetzungen

- **Binary:** `bin\BP-Analyzer.exe` im Skill-Ordner, win-x64, self-contained single file — **keine .NET-Runtime nötig**, Templates sind eingebettet.
- **Release v1.0.4 (Stand 04.11.2026):** [BP-Analyzer.zip](https://github.com/schirkan/bp-analyzer/releases/download/v1.0.4/BP-Analyzer.zip) ist das einzige Release-Asset (7.623.855 Bytes), für Windows x64. Es enthält `BP-Analyzer.exe` und die Beispielkonfiguration `bp-analyzer.config.json`. Kein Umbenennen erforderlich. Die SHA256-Prüfsumme steht im [Release-Text](https://github.com/schirkan/bp-analyzer/releases/tag/v1.0.4), nicht in einer separaten Datei. Bei späteren Releases Asset und Prüfsumme auf der Release-Seite prüfen, nicht aus alten Dateinamen ableiten.
- **Nur für `export`:** Blue Prism mit erreichbarer `AutomateC.exe`. Standard: `C:\Program Files\Blue Prism Limited\Blue Prism Automate\AutomateC.exe`. Priorität: `--automate-c=<pfad>` → `automateCPath` in `bp-analyzer.config.json` → Standardpfad. Die optionale Konfiguration wird zuerst neben der EXE, danach im aktuellen Arbeitsverzeichnis gesucht. `codegen` und `analyze` benötigen weder Blue-Prism-Installation noch Konfiguration.
- **Anmeldung** an die Zielumgebung: SSO oder Benutzer mit Passwort. Die Rolle muss den Export erlauben.
- **Schreibrechte** im Ablageverzeichnis.
- Für `codegen`: XML-Dateien mit Wurzelelement `process`.

## Sicherheitsregeln

1. **Lade das Binary nicht selbst herunter und baue es nicht selbst.** Fehlt `bin\BP-Analyzer.exe`, weise den Nutzer an, `BP-Analyzer.zip` aus den [Releases](https://github.com/schirkan/bp-analyzer/releases) herunterzuladen, die ZIP-Prüfsumme gegen den Release-Text abzugleichen und den Inhalt **manuell** im Skill-Ordner unter `bin\` abzulegen. Erwartet: `bin\BP-Analyzer.exe` und optional `bin\bp-analyzer.config.json`. Keine Umbenennung, keine separate Prüfsummendatei. Nenne Download-URL und Zielpfad; warte auf die Bestätigung.
2. **Bestätige Zielumgebung und Prozessnamen vor dem Export.** Der Export greift mit Credentials auf die Blue-Prism-Umgebung zu und schreibt Dateien an den Zielpfad.
3. **Entscheide `--overwrite` bewusst.** Standard ist `yes` — wiederholte Läufe überschreiben vorhandene Exporte. Bei wiederholten Analysen im selben Verzeichnis `--overwrite=no` verwenden und die Meldung `File already exists, skipping export` erwarten.
4. **Zugangsdaten bleiben aus Argumenten.** SSO bevorzugen. Passwörter nie in ein Kommando, einen Argument, eine Datei oder eine Ausgabe schreiben. Der interaktive Modus maskiert die Eingabe.
5. **Blue-Prism-XML enthält sensible Prozess- und Fachdaten.** Ergebnisse ausschließlich unter `analysis-records/` ablegen — der Ordner ist über die Root-`.gitignore` ausgeschlossen. Nichts committen, nichts hochladen, nichts in Chat-Ausgaben kopieren.
6. **Baue den generierten Code nicht.** Das `.vbproj` ist nur ein Hinweis auf die Codeform, kein Bauauftrag. Das `net48`-Targeting-Pack wird nicht vorausgesetzt.
7. **Behaupte nicht, der generierte Code bilde das Laufzeitverhalten ab.** UI-Automation, Object Calls und einige Stagetypen sind Platzhalter. Der Code ist eine statische, lesbare Darstellung der Struktur.
8. **Ändere oder baue das Tool nicht.** Bei abweichendem AutomateC-Installationsort ist die vorhandene Option `--automate-c=<pfad>` erlaubt; verwende einen vom Nutzer benannten Pfad. Die Konfiguration nicht ungefragt ändern. Fehlt `AutomateC.exe` am gewählten Pfad, melde die Voraussetzung und stoppe nur den Export, nicht unabhängige `codegen`-/`analyze`-Aufgaben.
9. **Keine Änderungen an Prozessen oder Sessions.** Der Export liest, er verändert die Blue-Prism-Umgebung nicht.

## Vorgehen

1. **Scope klären.** Prozessname, Zielumgebung, gewünschter Schritt. Bei Frage nach `analyze` klarstellen, dass es ein zusätzliches Dokument für Menschen erzeugt und für die Ablaufanalyse nicht nötig ist.
2. **Binary prüfen.** `bin\BP-Analyzer.exe` im Skill-Ordner. Fehlt es, den Nutzer anweisen und auf die Bestätigung warten.
3. **Nur vor `export`: AutomateC-Pfad prüfen.** Nutzerangabe über `--automate-c=<pfad>`, vorhandene optionale Konfiguration oder Standardpfad verwenden. Bei Unklarheit nachfragen; keine Installation oder Konfigurationsänderung vornehmen.
4. **Ablage festlegen.** Unter `analysis-records/<JJJJ-MM-TT>/<HHmmss-kurzer-bezug>/` die Unterordner `xml/`, `code/` und optional `sdd/` verwenden — identisch zur Nachweis-Ablage des Blue Prism Operator Agents.
5. **Export ausführen** und die Zusammenfassung prüfen: `Succeeded`, `Failed` und die Dateiliste. Jeder fehlgeschlagene Export wird einzeln mit Fehlermeldung gelistet.
6. **Codegen ausführen.** Anschließend prüfen, ob für **jede** XML-Datei eine `.vb`-Datei entstanden ist und ob die drei JSON-Dateien existieren. Fehlt eine Datei, wurde sie übersprungen — die Meldung `Error processing <datei>` steht nur im Text, der Exit-Code bleibt 0.
7. **Optional `analyze`.** Nur wenn der Nutzer eine Prozessdokumentation oder ein SDD wünscht.
8. **Bericht.** Ablauf in Stichpunkten, beteiligte Prozesse und Objects, gefundene Exception-Pfade, auffällige Stellen im generierten Code, Pfade der erzeugten Artefakte.

## Fehlerdiagnose

| Meldung | Ursache | Handlung |
|---|---|---|
| `Error: Process name is required.` | `--process` fehlt | Prozessnamen erfragen |
| `Error: --password *** required when --user is specified.` | Nur Benutzername übergeben | Zugangsdaten vervollständigen oder SSO nutzen |
| `Error: --user is required when --password *** specified.` | Nur Passwort übergeben | Beide Angaben machen |
| `Error: XML directory not found: <dir>` | `--xml` zeigt auf ein leeres oder falsches Verzeichnis | Verzeichnis aus dem vorherigen Export prüfen |
| `Invalid BluePrism XML file - root element must be 'process'` | Datei ist kein Prozess-Export, sondern z. B. ein BPA Object | Betroffene Datei identifizieren und aus der Codegen-Menge nehmen |
| `Error processing <datei>: <fehler>` | Einzelne Datei konnte nicht generiert werden | Meldung auswerten; die übrigen Dateien wurden erzeugt |
| Exit-Code 0 trotz `Error processing`-Meldung | `codegen` setzt den Exit-Code bei Dateifehlern nicht | Ausgabetext auswerten, nicht nur den Exit-Code |
| `File already exists, skipping export` | `--overwrite=no` und die Datei existiert | Vorhandenen Export prüfen oder mit `--overwrite=yes` bewusst überschreiben |
| `AutomateC.exe` startet nicht, keine XML erscheint | Falscher Installationspfad oder fehlende Rechte | Pfad und Rechte melden, nicht am Tool drehen |

## Implementierung

Reihenfolge und Aufrufe:

```powershell
$analyzer = (Resolve-Path -LiteralPath '.github\skills\bp-analyzer-cli\bin\BP-Analyzer.exe' -ErrorAction Stop).Path

# 1) XML exportieren
& $analyzer export "--process=Rechnungsverarbeitung" --output='analysis-records/2026-11-04/161500-rechnungsverarbeitung-analyzer/xml'
$exitCode = $LASTEXITCODE

# 2) Code erzeugen
& $analyzer codegen --xml='analysis-records/2026-11-04/161500-rechnungsverarbeitung-analyzer/xml' --output='analysis-records/2026-11-04/161500-rechnungsverarbeitung-analyzer/code'
$exitCode = $LASTEXITCODE

# 3) Optional: SDD
& $analyzer analyze --code='analysis-records/2026-11-04/161500-rechnungsverarbeitung-analyzer/code' --output='analysis-records/2026-11-04/161500-rechnungsverarbeitung-analyzer/sdd'
$exitCode = $LASTEXITCODE
```

Wichtige Details:

- Ohne Argument startet das CLI ein **interaktives Menü** (1 Export, 2 Codegen, 3 Analyze) und fragt fehlende Werte ab. Im nicht-interaktiven Betrieb immer die Form mit Argumenten verwenden.
- `codegen` löst Prozessaufrufe gegen das Verzeichnis aus `--xml=` auf, nicht gegen `<cwd>\xml`. Deshalb immer `--xml` explizit setzen.
- `analyze` erwartet `classes.json`, `dependencies.json` und `exceptions.json` im Verzeichnis aus `--code=` — also den Codegen-Schritt davor.
- `--exclude-exception-sources` ist eine kommagetrennte Liste; ohne Angabe gelten `ERGO_,Utility_`.
- Exit-Code `0` bedeutet nur, dass der Modus durchlief. Der Erfolg einzelner Dateien steht im Ausgabetext.
- Der Generator ist laut Projekt-TODO noch in Entwicklung. Dokumentiere den Ist-Stand mit Stand-Datum, statt ihn als stabil zu bezeichnen.

Vollständige Parameter- und Ausgabestruktur: `references/cli-reference.md`.

## Quelle / Source

- Das Tool lebt im Repository [**bp-analyzer**](https://github.com/schirkan/bp-analyzer). Ergänzungen, Korrekturen und Issues bitte dort einreichen. Das Binary für die lokale Ablage unter `bin\` stammt aus den [Releases](https://github.com/schirkan/bp-analyzer/releases) desselben Repositorys; der Agent lädt es nicht selbst herunter und baut es nicht selbst.
- Blue-Prism-CLI-Schalter für `AutomateC.exe`: siehe den Skill `bp-cli` und die offizielle Referenz [Blue Prism Command line options](https://documentation.blueprism.com/bp-7-4/en-us/helpCommandLine.htm).

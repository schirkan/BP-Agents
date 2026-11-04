---
name: bp-analyzer-cli-reference
description: "Vollständige Parameter- und Ausgabereferenz für den Skill bp-analyzer-cli: Modi export, codegen und analyze des BP-Analyzer, Ausgabeformate, Struktur des generierten VB.NET-Codes, JSON-Metadaten, Grenzen und bekannte Einschränkungen."
---

# BP Analyzer CLI — Parameter- und Ausgabereferenz

Diese Referenz dokumentiert das tatsächliche Verhalten des BP-Analyzer-CLI (Projektstand 04.11.2026). Beispiele verwenden fiktive Prozess- und Umgebungsnamen.

## 1. Binary und Aufruf

| Eigenschaft | Wert |
|---|---|
| Binary | `BP-Analyzer.exe`, win-x64, self-contained single file, trimmed, ca. 12,6 MB |
| .NET-Runtime | **nicht erforderlich** — Templates sind als Ressource eingebettet |
| Quellprojekt | `BP-Analyzer.csproj`, `TargetFramework net8.0`, AssemblyName `BP-Analyzer` |
| Publish | `dotnet publish BP-Analyzer.csproj -p:PublishProfile=BP-Analyzer-win-x64 -o publish` |

Im Repository existieren **keine Releases, keine Tags und keine CI-Workflows** — es gibt keinen fertigen Download und keine Release-Seite. Das Binary entsteht ausschließlich durch den oben gezeigten lokalen Publish, der das .NET-8-SDK vorausetzt. Der Agent führt diesen Publish nicht aus, sondern weist den Nutzer an.
| Template-Abhängigkeit | Handlebars.Net 2.1.6; `TemplateProvider` bevorzugt eine Datei auf der Platte, damit lokale Overrides weiter funktionieren |

Aufruf:

```
BP-Analyzer <modus> [optionen]
```

| Modus | Wirkung |
|---|---|
| `export` | Blue-Prism-Prozess als XML exportieren |
| `codegen` | Aus XML VB.NET-Code erzeugen |
| `analyze` | Aus den JSON-Metadaten ein SDD-Dokument erzeugen |
| `help`, `--help`, `-h` | Übersicht der Modi und Hinweis auf `<modus> --help` |

**Ohne Argumente** startet ein interaktives Menü (1 = Export, 2 = Codegen, 3 = Analyze); fehlende Parameter werden abgefragt. Im nicht-interaktiven Betrieb immer die Form mit Argumenten verwenden. Eine ungültige Menüauswahl liefert einen leeren Modus und führt zum Abbruch ohne Aktion.

## 2. Modus `export`

```
BP-Analyzer export --process=<name> [--output=<path>] [--user=<username>] [--password=***] [--overwrite=<yes|no>]
```

| Option | Pflicht | Standard | Wirkung |
|---|---|---|---|
| `--process=<name>` | ja | — | Name des zu exportierenden Prozesses. Wird in Anführungszeichen an `AutomateC.exe` durchgereicht. |
| `--output=<path>` | nein | `<cwd>\xml` | Zielverzeichnis. Wird bei Bedarf angelegt. |
| `--user=<username>` | bedingt | — | Blue-Prism-Benutzer. Ohne Angabe wird SSO verwendet. |
| `--password=***` | bedingt | — | Passwort. **Pflicht**, wenn `--user` gesetzt ist, und unzulässig ohne `--user`. |
| `--overwrite=<yes|no>` | nein | `yes` | Überschreiben vorhandener Exporte. |

Auch positionsgebundene Argumente werden akzeptiert (Prozess, Ausgabe, Benutzer, Passwort). Named Form verwenden.

### Zugangsdaten-Prüfung

| Eingabe | Reaktion | Exit |
|---|---|---|
| `--user` ohne `--password` | `Error: --password *** required when --user is specified.` | 1 |
| `--password` ohne `--user` | `Error: --user is required when --password *** specified.` | 1 |

Die Prüfung verhindert, dass der Export still auf SSO umschaltet. Im interaktiven Modus wird das Passwort maskiert eingelesen (Anzeige `*`, Rückschritt löscht).

### Ausgabe auf der Konsole

```text
=== Blue Prism Process Export (CLI mode) ===
Process: Rechnungsverarbeitung
Output directory: C:\Exports
Overwrite: yes

=== Export Result ===
Succeeded: 4
Failed: 0

Exported files:
  - C:\Exports\BPA Process - Rechnungsverarbeitung.xml
  - C:\Exports\BPA Object - Rechnungsverarbeitung_Rechnung.xml
```

Bei Fehlern zusätzlich:

```text
Failed exports:
  - C:\Exports\BPA Process - SonstigerProzess.xml: <Fehlermeldung>
```

Exit-Code: `0` wenn kein Export fehlschlug, sonst `1`.

### Was intern passiert

| Schritt | Verhalten |
|---|---|
| AutomateC-Aufruf | `C:\Program Files\Blue Prism Limited\Blue Prism Automate\AutomateC.exe` — **Pfad im Tool fest verdrahtet**, keine CLI-Option vorhanden |
| Argumente | mit Zugangsdaten: `/export "<name>" /user "<user>" "<password>"` — ohne: `/export "<name>" /sso` |
| Arbeitsverzeichnis | Temporärverzeichnis des Systems; die exportierte Datei wird von dort ins Zielverzeichnis verschoben |
| Erwartete Dateinamen | `BPA Process - <name>.xml` und `BPA Object - <name>.xml` |
| Vorhandene Temp-Dateien | werden vor dem Aufruf gelöscht |
| Abhängigkeiten | werden aus der exportierten XML gelesen und **rekursiv** exportiert |
| Interne Objekte | Namen mit Präfix `Blueprism.Automate` werden übersprungen |
| Vorhandene Zieldatei | bei `--overwrite=no`: `File already exists, skipping export` |

## 3. Modus `codegen`

```
BP-Analyzer codegen [--xml=<directory>] [--output=<directory>]
```

| Option | Standard | Wirkung |
|---|---|---|
| `--xml=<directory>` | `xml` | Verzeichnis mit Blue-Prism-XML-Dateien. Existiert es nicht: `Error: XML directory not found: <dir>`, Exit 1. |
| `--output=<directory>` | `code` | Zielverzeichnis für Code und JSON-Metadaten. Wird bei Bedarf angelegt. |

### Ausgabe auf der Konsole

```text
=== BluePrism VB.NET Code Generation ===
XML directory: xml
Output directory: code

Generated: code/Rechnungsverarbeitung.vb
Generated: code/Rechnungsverarbeitung_Rechnung.vb
Code generation completed!
```

Bei einem Dateifehler:

```text
Error processing xml/BPA Object - Defekt.xml: Invalid BluePrism XML file - root element must be 'process'
```

**Wichtig:** Ein Dateifehler beendet die Verarbeitung **nicht** — die übrigen Dateien werden weiter verarbeitet, und der Exit-Code bleibt `0`. Deshalb immer die `Generated:`-Zeilen zählen und mit der Zahl der XML-Dateien abgleichen.

### Erzeugte Dateien

| Datei | Inhalt |
|---|---|
| `<Prozess>.vb` | eine VB.NET-Klasse je XML-Datei; der Klassenname ist der Dateiname, bereinigt um ungültige Zeichen |
| `classes.json` | Metadaten der erzeugten Klassen |
| `dependencies.json` | Abhängigkeiten zwischen Prozessen, Business Objects und SubSheets |
| `exceptions.json` | aufgefangene und weitergereichte Exceptions |
| `BluePrism_Generated.vbproj` | Projektdatei für den generierten Code; wird **zuletzt** kopiert, damit alle Referenzen bekannt sind |

Prozessaufrufe werden gegen das Verzeichnis aus `--xml=` aufgelöst, nicht gegen `<cwd>\xml`. Deshalb `--xml` immer explizit setzen.

### Struktur des generierten VB.NET-Codes

| Element | Beschreibung |
|---|---|
| Basisklasse | `BP_Base`; die Basisklasse wird aus `Template_BP_Base.vb` erzeugt |
| Singleton | `Lazy(Of <Klasse>)` mit `Instance`-Property |
| Ablaufsteuerung | **GoTo-basiert** — Labels je Stage, `GoTo` zwischen den Stages |
| Exception-Handling | `On Error GoTo recover_onsuccess_Label` in Stages, die in einem Block oder auf einer Seite mit Recover-Stage liegen; `Resume resume_onsuccess_Label` für Resume-Stages |
| Datenitems | statische Variablen je Blue-Prism-Datenitem |
| Sichtbarkeit | in Prozessen ist nur die Main Page `public`, alle anderen Methoden `private`; in Objects können alle Methoden `public` sein (abhängig vom Parameter `published`) |
| Kommentare | Methoden-Kommentar wird aus `SubSheetInfo > narrative` extrahiert |
| Block-Stages | werden nicht als Stage gerendert; sie dienen der Berechnung, ob eine Stage in einem Block liegt |

### Grenzen des generierten Codes

- Der Code ist **syntaktisch gültig** und lässt sich kompilieren, ist aber **nicht ausführbar**. UI-Automation, Object Calls und einige Stagetypen sind als Platzhalter oder Stubs umgesetzt.
- Das Projektziel ist `net48`; ein Build setzt das .NET-Framework-4.8-Targeting-Pack voraus.
- Der Generator ist laut Projekt-TODO noch in Entwicklung. Aussagen zum Ist-Stand mit Datum versehen.

## 4. Modus `analyze`

```
BP-Analyzer analyze [--code=<json-dir>] [--output=<sdd-dir>] [--exclude-exception-sources=<prefix1,prefix2,...>]
```

| Option | Standard | Wirkung |
|---|---|---|
| `--code=<json-dir>` | `code` | Verzeichnis mit `classes.json`, `dependencies.json` und `exceptions.json` — also das Ergebnis von `codegen` |
| `--output=<sdd-dir>` | `sdd` | Zielverzeichnis für die SDD-Dateien |
| `--exclude-exception-sources=<prefixe>` | `ERGO_,Utility_` | kommagetrennte Präfixe von Exception-Quellen, die im SDD nicht erscheinen |

Der Modus liest die JSON-Metadaten und erzeugt daraus Markdown-Dokumente nach der Vorlage `SDD_Process_Template.md`.

**Reihenfolge beachten:** `analyze` braucht die Ausgabe von `codegen`. Ohne die drei JSON-Dateien im Verzeichnis aus `--code=` entsteht kein verwertbares Dokument.

Für die Ablaufanalyse ist dieser Modus **nicht erforderlich**. Er liefert ein menschlich lesbares Dokument, keine zusätzlichen Erkenntnisse über den Ablauf.

## 5. Zusammenspiel der Modi

```powershell
$analyzer = (Resolve-Path -LiteralPath '.github\skills\bp-analyzer-cli\bin\BP-Analyzer.exe' -ErrorAction Stop).Path
$base = 'analysis-records/2026-11-04/161500-rechnungsverarbeitung-analyzer'

& $analyzer export  --process='Rechnungsverarbeitung' --output="$base/xml"
$exitCode = $LASTEXITCODE

& $analyzer codegen --xml="$base/xml"  --output="$base/code"
$exitCode = $LASTEXITCODE

# Optional
& $analyzer analyze --code="$base/code" --output="$base/sdd"
$exitCode = $LASTEXITCODE
```

| Prüfpunkt | erwartet |
|---|---|
| nach `export` | `Succeeded: N` ohne `Failed:`-Block; je Abhängigkeit eine XML-Datei |
| nach `codegen` | je XML eine `.vb`-Datei; `classes.json`, `dependencies.json`, `exceptions.json` vorhanden |
| nach `analyze` | eine SDD-Datei je Prozess |

## 6. Bekannte Einschränkungen

| Einschränkung | Auswirkung |
|---|---|
| `AutomateC.exe`-Pfad fest verdrahtet | Bei abweichendem Installationsort scheitert der Export. Es gibt keine CLI-Option zum Überschreiben. Abweichung melden, das Tool nicht ändern. |
| Export ohne Datenbankzugriff | Der Export läuft ausschließlich über die Blue-Prism-CLI. Eine direkte Datenbankabfrage ist nicht möglich. |
| Exit-Code 0 bei Dateifehlern im `codegen` | Erfolg muss anhand der Ausgabe geprüft werden, nicht am Exit-Code. |
| Kein `version`-Befehl | Das CLI kennt keinen Versionsschalter. Der Stand ergibt sich aus dem Stand des Quell-Repositories. |
| Generierter Code nicht ausführbar | Der Code ist eine statische Darstellung der Struktur, keine Abbildung des Laufzeitverhaltens. |
| Passwort als Argument | `--password=***` ist in der Prozessliste sichtbar. SSO oder den interaktiven Modus verwenden. |

## 7. Quellen

- Projekt und Binary: [**bp-analyzer**](https://github.com/schirkan/bp-analyzer)
- Blue-Prism-CLI-Schalter für `AutomateC.exe`: Skill `bp-cli` und [Blue Prism Command line options](https://documentation.blueprism.com/bp-7-4/en-us/helpCommandLine.htm)

---
name: bp-cli
description: "Use for Blue Prism command-line tasks involving Automate.exe or AutomateC.exe: command-line switches, authentication, database connections, process and release import/export, schedules, work queues, runtime control, and diagnostics."
argument-hint: "Blue Prism CLI task, installed version, target environment, and the intended read or change."
---

# Blue Prism CLI

## Zweck

Dokumentiert und unterstützt die Blue-Prism-Command-Line-Optionen von `Automate.exe` und `AutomateC.exe`. Der Skill hilft beim Ermitteln des passenden Aufrufs, beim Einordnen der Ausgabe und beim sicheren Umgang mit Änderungen an Blue-Prism-Umgebungen.

## Input

| Feld | Pflicht | Bedeutung |
|---|---|---|
| `task` | ja | Gewünschte Abfrage oder Aktion, z. B. Prozesse auflisten, Release importieren oder Laufstatus prüfen. |
| `version` | empfohlen | Blue-Prism-Version des installierten CLI-Binaries. Schalter können sich zwischen Versionen unterscheiden. |
| `target` | bei Verbindung/Aktion | Zielumgebung, benannte Datenbankverbindung, Ressource oder betroffenes Objekt. Keine Passwörter oder Tokens übermitteln. |
| `authorization` | bei Änderung | Bestätigung, dass der Aufruf für die Zielumgebung autorisiert ist. Änderungen brauchen eine ausdrückliche Bestätigung. |

## Output

Eine passende CLI-Auswahl und ein PowerShell-Aufruf mit:

- `Automate.exe` oder `AutomateC.exe` und dem passenden Switch,
- Voraussetzungen und versionsabhängigen Einschränkungen,
- erwarteter Ausgabe sowie Hinweisen zum Exit-Code,
- möglichen Nebenwirkungen und erforderlicher Bestätigung für Änderungen.

Wenn die installierte Version oder die Bedeutung eines Parameters unklar ist, erst die lokale `/help`-Ausgabe prüfen und keine unbelegten Parameter annehmen.

## Voraussetzungen

- Blue Prism ist installiert; CLI-Binaries liegen typischerweise unter `C:\Program Files\Blue Prism Limited\Blue Prism Automate\`.
- Die ausführende Identität besitzt die für die Aktion erforderliche Blue-Prism-Rolle und Datenbankberechtigungen.
- Für Verbindungsvorgänge ist eine konfigurierte Datenbankverbindung oder ein expliziter Verbindungsname erforderlich.
- Anmeldung bevorzugt über `/sso`, sofern verfügbar. `/user <name> <password>` kann das Kennwort als Kommandozeilenargument offenlegen; niemals echte Zugangsdaten in Beispiele, Chat, Skripte, Logs oder Commit-Historie schreiben.

## Vorgehen

1. Version, Zielumgebung und Absicht (nur lesen oder ändern) feststellen.
2. Bei nicht eindeutiger Binary-Version oder unbekannten Optionen die lokale Hilfe des betreffenden Binaries ansehen:

   ```powershell
   & 'C:\Program Files\Blue Prism Limited\Blue Prism Automate\AutomateC.exe' /help
   $exitCode = $LASTEXITCODE
   ```

   Für `Automate.exe` den entsprechenden Pfad verwenden. Hilfeausgabe und unterstützte Optionen können je nach Version abweichen.
3. Die [CLI-Referenz](./references/cli-reference.md) für Binary-Unterschiede, Syntax, Switches und Auswirkungen konsultieren.
4. In PowerShell den Aufrufoperator `&` und einen konkreten Binary-Pfad verwenden. Argumente mit Leerzeichen quoten; keine Passwörter in Beispiele oder Logs übernehmen.
5. Vor jedem zustandsändernden oder destruktiven Aufruf Ziel, betroffene Objekte, erwartete Wirkung und Wiederherstellbarkeit erläutern und ausdrücklich bestätigen lassen. Ohne Bestätigung nur lesende Diagnose durchführen.
6. Ausgabe auf stdout und stderr sowie `$LASTEXITCODE` unmittelbar auswerten. Bei schreibenden Aktionen zusätzlich den resultierenden Zustand in Blue Prism verifizieren; Erfolgstext allein ist kein ausreichender Nachweis.

## Implementierung

Dieser Skill führt selbst keine CLI aus. Er beschreibt die mit Blue Prism installierten Werkzeuge. `Automate.exe` ist die grafische Anwendung; `AutomateC.exe` ist das Kommandozeilenwerkzeug und gibt Meldungen über die Konsole aus. Eine Auswahl der dokumentierten Switches und ihre Risiken stehen in der [CLI-Referenz](./references/cli-reference.md).

## Quellen

- Blue Prism, [Command line options (7.4)](https://documentation.blueprism.com/bp-7-4/en-us/helpCommandLine.htm) — offizielle Referenz.
- [AutomateC.exe — Blue Prism 7.5.1 CLI Reference](https://github.com/schirkan/bp-git/blob/main/context/bp-cli-reference-7.5.1.md) — ergänzende, versionsgebundene Referenz; Angaben nicht ungeprüft auf andere Versionen übertragen.

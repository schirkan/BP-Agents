# Blue Prism CLI – Referenz

Diese Referenz behandelt `Automate.exe` und `AutomateC.exe`. Sie fasst die offizielle Blue-Prism-Dokumentation für Version 7.4 und ergänzende Angaben zu `AutomateC.exe` 7.5.1 zusammen. Die angebotenen Schalter und das genaue Verhalten können sich je Version unterscheiden. Die lokale Hilfe des installierten Binaries und die offizielle Dokumentation der passenden Blue-Prism-Version haben Vorrang.

## Binaries unterscheiden

| Binary | Zweck | Ausgabe |
|---|---|---|
| `Automate.exe` | Grafische Blue-Prism-Anwendung; unterstützt Command-Line-Switches und kann UI bzw. Dialoge anzeigen. | Meldungen können in der Anwendung erscheinen; nicht als headless CLI voraussetzen. |
| `AutomateC.exe` | Kommandozeilenwerkzeug für Blue-Prism-Operationen. | Meldungen über stdout/stderr; Exit-Code direkt nach dem Aufruf erfassen. |

Üblicher Installationsordner:

```text
C:\Program Files\Blue Prism Limited\Blue Prism Automate\
```

Der tatsächliche Installationspfad und die unterstützten Switches sind auf dem Zielsystem zu prüfen.

## Syntax und Hilfe

Die dokumentierten Optionen verwenden in der Regel `/switch`-Syntax. Werte folgen dem Switch und können je nach Option mehrere Argumente umfassen. Die konkreten Regeln für Reihenfolge, Quotes und zusammengesetzte Switches sind versionsabhängig.

```powershell
$cli = 'C:\Program Files\Blue Prism Limited\Blue Prism Automate\AutomateC.exe'
& $cli /help
$exitCode = $LASTEXITCODE
```

Für `Automate.exe` den Pfad zum entsprechenden Binary verwenden. Hilfeausgabe und Exit-Code erfassen; ein erfolgreiches `/help` bestätigt nicht, dass ein späterer Aufruf für die angegebene Version gültig ist. Bei Unsicherheit zuerst die lokale Hilfe und danach die offizielle Doku der installierten Version heranziehen.

## Anmeldung und Verbindungen

| Switch | Syntax | Zweck / Hinweise |
|---|---|---|
| `/user` | `<name> <password>` | Blue-Prism-Anmeldung. Das Passwort steht dabei als Prozessargument und kann für andere lokale Prozesse oder in Diagnoseartefakten sichtbar sein. Nicht in Skripte, Logs oder Beispielausgaben übernehmen. |
| `/sso` | — | Single Sign-On mit dem aktuellen Windows-Benutzer, wenn es in der Umgebung unterstützt und konfiguriert ist. Bevorzugen, wenn möglich. |
| `/dbconname` | `<connection-name>` | Ausgewählte Blue-Prism-Datenbankverbindung. Den Namen aus der lokalen Konfiguration übernehmen. |
| `/resource` | `<target>` | Resource-PC-Ziel für entsprechende Operationen. |
| `/port` | `<portnumber>` | Port für Resource-PC-Verbindungen. |

Die Zusatzreferenz zu 7.5.1 berichtet, dass eine Standard-Datenbankverbindung ohne `/dbconname` verwendet werden kann. Das ist versions- und installationsabhängig; für reproduzierbare Aufrufe die gewünschte Verbindung explizit angeben, sofern der Befehl dies unterstützt.

## Switch-Katalog (ergänzende Referenz: 7.5.1)

Die folgenden Switches sind aus der verlinkten 7.5.1-Referenz zusammengefasst und nicht als vollständige oder versionsübergreifend garantierte Liste zu verstehen. Unbedingt die lokale `/help`-Ausgabe prüfen.

### Datenbank und Konfiguration

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/createdb` | `<password>` | Datenbank erstellen; die ergänzende Referenz beschreibt das Verhalten als Drop und Neuerstellung. Destruktiv. |
| `/replacedb` | `<password>` | Datenbanktabellen ersetzen bzw. neu aufbauen. Destruktiv. |
| `/upgradedb` | `<password>` | Datenbank aktualisieren. Ändert das Schema. |
| `/getdbscript` | SQL-Skript auf stdout | Datenbank-Erstellungsskript ausgeben. |
| `/maxdbver` | — | Maximale Datenbankversion beim Erstellen/Aktualisieren verwenden. |
| `/setbpserver` | `<host> <port>` | Blue-Prism-Server-Verbindung konfigurieren. |
| `/ag` | `<host> <port>` | Availability-Group-Verbindungsoption (Zusatzreferenz). |
| `/agport` | `<host> <port>` | Port des AG-Listeners (Zusatzreferenz nennt 1433 als Standard). |
| `/showdbconfig` | — | Datenbank-Konfigurationsdialog anzeigen. |

### Prozesse und Releases

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/listprocesses` | — | Prozessnamen auflisten. |
| `/import` | `<filespec> [/forceid {new\|<guid>}] [/overwrite]` | Prozessobjekt importieren; kann bestehende Inhalte überschreiben. |
| `/export` | `<processname>` | Prozess als XML exportieren. |
| `/publish` / `/unpublish` | `<processname>` | Prozess veröffentlichen bzw. Veröffentlichung zurücknehmen. |
| `/publishws` | `<processname> <servicename> [options]` | Prozess als Web-Service veröffentlichen. |
| `/unpublishws` | `<processname>` | Web-Service-Veröffentlichung zurücknehmen. |
| `/refreshdependencies` | `{force}` | Prozessabhängigkeiten neu aufbauen. |
| `/genivbowrapper` | `<name>` | Wrapper für ein internes Business Object erzeugen. |
| `/importrelease` | `<filespec>` | Release-Datei importieren; 7.5.1-Referenz nennt `.bprelease` und weist auf `.bpskill`-Support in der 7.x-Dokumentation hin. Ändert die Umgebung. |
| `/exportpackage` | `<packagename> [/release <releasename>]` | Package oder Release als XML exportieren. |

Vor Import, Überschreiben oder Veröffentlichung Ziel und Objektbestand verifizieren. `/importrelease` nicht mit einem reinen Export- oder Lesevorgang verwechseln.

### Work Queues

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/createqueue` | `<keyfield> <running> <maxattempts>` | Work Queue erstellen. |
| `/exportqueue` | `<filespec> [/queuefilter <filtername>] [/clearexported]` | Queue exportieren; `/clearexported` kann exportierte Items löschen. |
| `/deletequeue` | — | Work Queue löschen. Destruktiv. |
| `/queueclearworked` | `/queuename <name> [/age <age>]` | Bearbeitete bzw. Exception-Items löschen. Destruktiv. |

### Schedules

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/startschedule` | `[/schedule <name>\|...]` | Schedule(s) starten; löst geplante Automatisierungen aus. |
| `/deleteschedule` | `[/schedule <name>\|...]` | Schedule(s) löschen. |
| `/viewschedtimetable` | `<name>` oder Zeitraum, optional `/schedule` und `/format {csv\|txt}` | Schedule-Timetable anzeigen. |
| `/viewschedreport` | `<name>` oder Zeitraum, optional `/schedule` und `/format {csv\|txt}` | Schedule-Report anzeigen. |

### Environment-Variablen und Archive

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/setev` | `<name> <datatype> <value> <description>` | Environment-Variable setzen. |
| `/deleteev` | `<name>` | Environment-Variable löschen. |
| `/archive` | Filter wie `/from`, `/to`, `/age`, `/process`; optional `/delete` | Logs archivieren; `/delete` kann Quelldaten löschen. |
| `/restorearchive` | — | Archivierte Logs wiederherstellen. |
| `/setarchivepath` | `<path>` | Archivpfad konfigurieren. |

### Runtime und Ressourcen

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/run` | `<processname> [/startp <xml>]` | Prozess starten. Führt Geschäftsautomatisierung aus. |
| `/status` | `<sessionid>` | Session-Status abfragen. |
| `/getlog` | `<sessionid>` | Session-Log abrufen. |
| `/requeststop` | `<sessionid\|sessionnumber>` | Laufende Session stoppen. |
| `/getauthtoken` | `[/process <processname>]` | Authentifizierungs-Token abrufen. |
| `/getbod` | `<objectname>` | Business-Object-Definition abrufen. |
| `/getauditlog` | — | Audit-Log abrufen. |
| `/resourcestatus` | `<resourcename> <limit> <type>` | Resource-Sessions abfragen. |
| `/wslog` | `on\|off` | Web-Service-Logging umschalten. |
| `/poolcreate` | `/pool <name>` | Resource Pool erstellen. |
| `/pooldelete` | `/pool <name>` | Resource Pool löschen. |
| `/pooladd` | `/pool <name> [/resource <name>]` | Resource zum Pool hinzufügen. |
| `/poolremove` | `[/resource <name>]` | Resource aus Pool entfernen. |

### Server, Authentifizierung und Credentials

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/serverconfig` | `<name> <connection> <port>` | Blue-Prism-App-Server-Konfiguration erstellen/aktualisieren. |
| `/connectionmode` | `<value>` | Verbindungsmodus konfigurieren; genaue Werte versionsabhängig. |
| `/encryptionscheme` | `<name> [<method>]` | Verschlüsselungsschema konfigurieren. |
| `/ordered` | `<value>` | Ordered Sessions konfigurieren. |
| `/setallowanonresources` | `<value>` | Anonyme Resources konfigurieren. |
| `/serviceaccount` | `<clientid> <clientsecret>` | Service-Account konfigurieren. Secret nicht in Kommandozeile/Logs exponieren. |
| `/resetdefaultadminpassword` | `<new password>` | Standard-Administratorpasswort zurücksetzen. Sensible Änderung. |
| `/setactivedirectoryauth` | `<flag>` | Active-Directory-Authentifizierung konfigurieren. |
| `/setactivedirectorygroupbasedroles` | `<flag>` | Gruppenbasierte Rollen konfigurieren. |
| `/mapactivedirectorygrouptorole` | `<role> <group SID> <group DN>` | AD-Gruppe einer Rolle zuordnen. |
| `/mapauthenticationserverusers` | `<inputcsv> <outputcsv>` | Benutzer zwischen Blue Prism und Authentication Server zuordnen. |
| `/getblueprismtemplateforusermapping` | `<outputcsv>` | Vorlage für Blue-Prism-Benutzerzuordnung erzeugen. |
| `/getauthenticationservertemplateforusermapping` | `<outputcsv>` | Vorlage für Authentication-Server-Benutzerzuordnung erzeugen. |
| `/setkerberosrealm` | `<kerberosrealm>` | Kerberos-Realm konfigurieren. |
| `/forcentlm` | `<flag>` | NTLM-Verhalten konfigurieren. |
| `/createcredential` | `<credname> <username> <password> [options]` | Blue-Prism-Credential erstellen. Zugangsdaten nicht als echte Beispielwerte verwenden. |
| `/updatecredential` | `<credname> [options]` | Credential aktualisieren. |
| `/setcredentialproperty` | `<credname> <propertyname> <propertyvalue>` | Credential-Eigenschaft erstellen/aktualisieren. |

### Lizenzen, Reports und Verschlüsselung

| Switch | Syntax / Zweck | Wirkung |
|---|---|---|
| `/license` | `<licensefile>` | Lizenz hinzufügen. |
| `/removelicense` | `<licensefile>` | Lizenz entfernen. |
| `/report` | `<filespec>` | Report ausgeben. |
| `/rolereport` | `<filespec>` | Rollenberechtigungs-Report ausgeben. |
| `/elementusage` | `<filespec>` | Element-Usage-Report ausgeben. |
| `/fontimport` | `<filespec>` | Font importieren. |
| `/setencrypt` | `<encrypter-name>` | Verschlüsselungs-Provider setzen. |
| `/resetencrypt` | — | Verschlüsselungs-Provider zurücksetzen. |
| `/reencryptdata` | `[/batchsize <size>] [/maxbatches <number>]` | Daten erneut verschlüsseln. |
| `/configencrypt` | `[default\|thumbprint] [/forceconfigencrypt]` | Verschlüsselung der Konfiguration setzen. |

## Sicherer Umgang mit Nebenwirkungen

Diese Aufrufe können produktive Daten oder laufende Automatisierungen verändern. Vor ihrer Ausführung müssen Zielsystem, Datenbankverbindung, betroffene Objekte und Wirkung eindeutig sein und ausdrücklich bestätigt werden:

- **Datenbank:** `/createdb`, `/replacedb`, `/upgradedb`
- **Import/Veröffentlichung:** `/import`, `/importrelease`, `/publish`, `/unpublish`, `/publishws`, `/unpublishws`
- **Queues/Schedules:** `/createqueue`, `/deletequeue`, `/queueclearworked`, `/startschedule`, `/deleteschedule`
- **Runtime:** `/run`, `/requeststop`
- **Konfiguration und Zugänge:** `/setev`, `/deleteev`, Credential-, Auth-, Lizenz- und Verschlüsselungsoptionen
- **Archive:** `/archive` besonders mit `/delete`, `/restorearchive`, `/setarchivepath`

Keine Auswahl oder Kennung raten. Für nicht bestätigte oder mehrdeutige Ziele nur lesende Abfragen verwenden. Nach Änderungen den Zustand über Blue Prism oder eine geeignete Abfrage nachprüfen.

## Beispiel: lesende Abfrage

Nur wenn die lokale Version den Switch unterstützt und SSO konfiguriert ist:

```powershell
$cli = 'C:\Program Files\Blue Prism Limited\Blue Prism Automate\AutomateC.exe'
& $cli /sso /dbconname 'BP-Development' /listprocesses
$exitCode = $LASTEXITCODE
```

`BP-Development` ist ein Platzhalter. Den konfigurierten Verbindungsnamen verwenden. Ausgabe, stderr und Exit-Code getrennt bewerten.

## Quellen

- Offiziell: [Blue Prism 7.4 — Command line options](https://documentation.blueprism.com/bp-7-4/en-us/helpCommandLine.htm)
- Ergänzend: [Blue Prism 7.5.1 CLI Reference](https://github.com/schirkan/bp-git/blob/main/context/bp-cli-reference-7.5.1.md)

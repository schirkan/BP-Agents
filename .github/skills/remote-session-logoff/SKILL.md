---
name: remote-session-logoff
description: "Verwende diesen Skill, um eine angemeldete Benutzersession auf einem entfernten Windows-System abzumelden: Session-Verifikation, Credential-Handhabung, Bestätigungspflicht, logoff vs. rwinsta, Nachkontrolle und Fehlerdiagnose. Destruktiver Eingriff — Ausführung nur nach ausdrücklicher Bestätigung."
argument-hint: "Hostname oder IP des Ziel-Systems, Session-ID, erwarteter Benutzername zur Absicherung gegen Verwechslung."
---

# Remote Session Logoff

## Zweck

Meldet eine interaktive Benutzersession auf einem entfernten Windows-System ab. Der Benutzer wird ausgeloggt, alle in dieser Session laufenden Prozesse werden beendet, die Session-ID wird freigegeben.

Der Eingriff ist **destruktiv und nicht umkehrbar**. Es gibt keinen Wiederherstellungsweg für eine beendete Session: offene Dokumente, nicht gespeicherte Arbeit und laufende Prozesse gehen verloren. Der Skill ist deshalb so gebaut, dass er vor der Ausführung verifiziert, explizit bestätigen lässt und danach nachkontrolliert.

Die **eigene Session auf dem Zielsystem** darf abgemeldet werden — das Freigeben einer eigenen RDP-Sitzung ist ein regulärer Anwendungsfall. Geschützt ist ausschließlich die lokale Sitzung, in der der Aufruf selbst ausgeführt wird.

Kernwerkzeug ist `logoff` ( orderly Abmeldung). `rwinsta` steht als harte Alternative zur Verfügung, wird aber nur auf ausdrückliche Weisung verwendet.

## Input

| Feld | Pflicht | Bedeutung |
|---|---|---|
| `host` | ja | Hostname oder IP des Ziel-Systems. Kein Pfad, kein URL-Schema. |
| `session_id` | ja | Numerische Session-ID. **Nie raten** — immer aus einer aktuellen `quser`-Ausgabe ableiten. |
| `expected_username` | ja | Benutzername, der in der Ziel-Session erwartet wird. Dient als Absicherung gegen Verwechslung. Bei Abweichung wird nicht ausgeführt. |
| `mode` | nein | `logoff` (Default, orderly) oder `rwinsta` (hartes Beenden). `rwinsta` nur auf ausdrückliche Weisung. |
| `credentials` | ja | Anmeldedaten für das Ziel. Passwort niemals im Klartext in Argumente, Ausgaben, Logs oder Dateien schreiben. |

## Output

| Feld | Bedeutung |
|---|---|
| `host` | Zielsystem |
| `session_id` / `session_name` | die abgemeldete Session |
| `username` | Benutzer der Session |
| `action` | `logoff` oder `rwinsta` |
| `exit_code` | Rückgabewert des Befehls |
| `status` | `success`, `failed`, `not_found`, `mismatch`, `declined`, `verification_failed` |
| `verification` | Ergebnis der Nachkontrolle: Session in der erneuten `quser`-Ausgabe nicht mehr enthalten |

## Voraussetzungen

- **Administratorrechte auf dem Ziel.** `logoff` und `rwinsta` sind Administratorwerkzeuge; ein Standardkonto reicht nicht.
- **Passwort des Administrators** ist Voraussetzung — die Abfrage authentifiziert sich gegenüber dem Ziel, nicht gegenüber dem lokalen System.
- **RPC-Erreichbarkeit:** TCP 135 plus dynamischer Portbereich (TCP 49152–65535) muss zum Ziel freigegeben sein. Nur 135 reicht nicht.
- Die Varianten `logoff /server:<host>` und `rwinsta /server:<host>` haben **keine eigene Credential-Option**. Das Passwort muss vorab über `cmdkey` oder `net use` für das Ziel hinterlegt werden, oder die CIM-Variante `Win32_Logoff` wird über WinRM mit `-Credential` genutzt (bevorzugt, weil das Passwort nicht in der Kommandozeile steht).
- Der Aufruf erfolgt aus einer **erhöhten** Shell (Administrator-Kontext), sonst schlägt die RPC-Authentifizierung fehl.
- **Selbstbeendigung beachten:** Läuft der ausführende Prozess in derselben Windows-Session, die abgemeldet werden soll, beendet `logoff` ihn mit. Der Befehl liefert dann keinen Exit-Code mehr, und eine WinRM-Verbindung bricht ab. Das ist **kein Fehlschlag**, sondern das erwartete Verhalten bei der Abmeldung der eigenen Session. In diesem Fall ausschließlich über die Nachkontrolle verifizieren und das Ergebnis berichten — den Befehl nicht wiederholen, weil die Session-ID inzwischen neu vergeben sein kann.

## Sicherheitsregeln

Diese Regeln sind bindend. Der Skill bricht ab, wenn eine davon nicht erfüllt ist.

1. **Bestätigungspflicht.** Vor der Ausführung das Ziel mit Host, Benutzername, Session-ID, Sessionname, Zustand und Auswirkung benennen und eine ausdrückliche Bestätigung einholen. Ohne Bestätigung nur die Verifikation durchführen und berichten.
2. **Session-ID niemals raten.** Immer zuerst `quser /server:<host>` ausführen und die ID daraus übernehmen. Eine veraltete oder geratene ID kann die falsche Person treffen.
3. **Benutzername abgleichen.** Stimmt der Benutzer der gefundenen Session nicht mit `expected_username` überein, wird **nicht** ausgeführt. Stattdessen die tatsächlich gefundenen Sessions melden und nachfragen.
4. **Nur die lokale Aufruf-Sitzung ist geschützt.** Die Session, in der der Aufruf auf dem **aufrufenden** System läuft, wird nie abgemeldet. Die eigene Session auf dem **Zielsystem** ist ausdrücklich erlaubt und folgt dem normalen Ablauf. Voraussetzung dafür ist, dass Zielsystem und aufrufendes System verschieden sind.
5. `logoff` vor `rwinsta`. `rwinsta` beendet die Session hart, ohne orderly Shutdown. Datenverlust in Anwendungen ist möglich. `rwinsta` nur bei ausdrücklicher Weisung.
6. **Nur die angeforderte Session.** Kein Abräumen mehrerer Sessions, keine Schleife über alle angemeldeten User. Jede zusätzliche Session braucht eine eigene Bestätigung.
7. **Passwörter bleiben aus dem Output.** Weder in Befehlsargumenten noch in Fehlermeldungen, Protokollen oder Dateien. `Get-Credential` bevorzugen; das Passwort wird interaktiv abgefragt und nie protokolliert.
8. **Immer nachkontrollieren.** Nach dem Befehl erneut `quser /server:<host>` ausführen und prüfen, dass die Session wirklich verschwunden ist. Ein Exit-Code 0 allein ist kein Nachweis.
9. **Interaktive Prozesse vorher prüfen.** Läuft in der Session eine Anwendung mit ungespeicherter Arbeit (Editor, Office-Anwendung, Client mit Hintergrundverarbeitung), wird diese ohne Vorwarnung beendet. Vorher auf laufende Prozesse prüfen und den Nutzer darauf hinweisen.
10. **Keine Ausführung bei unklarem Ziel.** Host, Session oder Benutzer nicht eindeutig → nur berichten, nichts ausführen.

## Vorgehen

1. **Scope klären.** Host, betroffener Benutzer und gewünschte Wirkung. Rückfrage stellen, wenn etwas unklar ist. Bei einem Terminal-Server mit vielen Sessions den Benutzerkreis eingrenzen.
2. **Session verifizieren, ohne etwas zu verändern:**

```powershell
& quser /server:'srv-host-01'
$exitCode = $LASTEXITCODE
```

3. **Gefundene Session gegen `expected_username` prüfen.** Bei Abweichung abbrechen und die tatsächliche Session-Liste melden. Detail-Auswertung: Skill `remote-logged-on-users`.
4. **Bestätigung einholen.** Angaben: Host, Benutzername, Session-ID, Sessionname, Zustand, Verbindungstyp, Client, Wirkung, gewählter Modus. Erst bei ausdrücklicher Zustimmung fortfahren. Ist die Ziel-Session die eigene Sitzung des anfragenden Users, wird das in der Bestätigung ausdrücklich benannt, damit die Freigabe eindeutig ist.
5. **Ausführen.** Empfohlene Variante mit sauberem Credential-Handling über WinRM:

```powershell
$cred = Get-Credential
Invoke-CimMethod -ClassName Win32_LogOff -MethodName LogOff \
    -Arguments @{ SessionId = 2 } \
    -ComputerName 'srv-host-01' -Credential $cred
```

Alternativ mit den Bordmitteln, wenn die Verbindung zum Ziel bereits authentifiziert ist:

```powershell
& logoff 2 /server:'srv-host-01'
$exitCode = $LASTEXITCODE
```

6. **Ergebnis bewerten.** Exit-Code bzw. `ReturnValue` auswerten, stderr lesen. `ReturnValue = 0` bedeutet Erfolg, `2` = Session nicht gefunden, `5` = Zugriff verweigert, `9` = allgemeiner Fehler. Details in `references/command-reference.md`.
7. **Nachkontrollieren.**

```powershell
& quser /server:'srv-host-01'
$exitCode = $LASTEXITCODE
```

Die Session darf in der Ausgabe nicht mehr auftauchen. Taucht sie weiterhin auf, ist der Eingriff fehlgeschlagen — das berichten, nicht wiederholen.
8. **Bericht.** Host, Benutzer, Session-ID, ausgeführter Befehl, Exit-Code, Verifikationsergebnis und ausgelöste Warnungen nennen.

## Fehlerdiagnose

| Meldung | Ursache | Handlung |
|---|---|---|
| `Access is denied.` | Fehlende Administratorrechte auf dem Ziel oder fehlende/falsche Credentials | Credentials prüfen; Adminrechte des verwendeten Kontos auf dem Ziel bestätigen. Nicht die lokale Shell hochstufen, wenn das Ziel die Ursache ist |
| `The system could not find the session specified.` | Session-ID existiert nicht (bereits abgemeldet) oder ist veraltet | `quser /server:<host>` neu ausführen und aktuelle ID verwenden |
| `RPC server unavailable.` | TCP 135 oder dynamischer RPC-Portbereich blockiert | Firewall des Ziels prüfen, Portbereich freigeben |
| `The network path was not found.` | Namensauflösung oder Routing fehlgeschlagen | Hostname prüfen, IP testen |
| `ReturnValue = 5` (CIM) | Zugriff verweigert | Credentials und Rechte auf dem Ziel prüfen |
| `ReturnValue = 2` (CIM) | Session nicht gefunden | Session-Liste neu abfragen |
| Exit 0, Session weiterhin gelistet | Falsche Session getroffen oder Logoff wirkte nicht | ID gegenprüfen, Ergebnis melden, nicht blind wiederholen |
| Befehl bricht ohne Rückgabe ab, keine Fehlermeldung | Der ausführende Prozess lief in der Session, die abgemeldet wurde | Kein Fehlschlag. Über `quser /server:<host>` prüfen, ob die Session verschwunden ist. Befehl nicht wiederholen |
| `Access is denied` trotz korrekter Credentials | Das Ziel verlangt eine andere Domäne oder ein anderes Konto | Zielkontext prüfen: mit welchem Konto wird sich am Ziel authentifiziert? `cmdkey /list` bzw. die Credential des Aufrufers gegenprüfen |

## Implementierung

Bevorzugte Reihenfolge:

1. **`Win32_LogOff` über CIM** mit `-Credential`. Der einzige Weg, bei dem das Passwort sauber interaktiv bleibt und nicht in der Kommandozeile steht.
2. **`logoff <id> /server:<host>`** — orderly Abmeldung, wenn die Verbindung zum Ziel bereits authentifiziert ist (`cmdkey` bzw. `net use`).
3. **`rwinsta <id> /server:<host>`** — harter Reset, nur auf ausdrückliche Weisung.

Alle drei nutzen RPC; der CIM-Weg zusätzlich WinRM. Skript-Vorlagen, Credential-Handhabung und die vollständige Befehlsreferenz stehen in `references/command-reference.md`.

## Quelle / Source

- **Befehlsreferenz:** Microsoft Learn — [logoff](https://learn.microsoft.com/windows-server/administration/windows-commands/logoff), [rwinsta](https://learn.microsoft.com/windows-server/administration/windows-commands/rwinsta), [query user](https://learn.microsoft.com/windows-server/administration/windows-commands/query-user)
- **Pflege:** Dieser Skill wird im Repository [**BP-Agents**](https://github.com/schirkan/BP-Agents) gepflegt (Pfad: `.github/skills/remote-session-logoff/`). Ergänzungen und Korrekturen bitte dort als PR einreichen.

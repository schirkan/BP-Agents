---
name: remote-logged-on-users
description: "Verwende diesen Skill, um angemeldete Benutzer auf entfernten Windows-Servern und Arbeitsplätzen abzufragen: qwinsta/query user, Session-Zustände (aktiv, getrennt), PowerShell-Alternativen, Filter, Ausgabeformat und Fehlerdiagnose. Enthält die Grenzen der einzelnen Methoden."
argument-hint: "Hostname oder IP des Ziel-Servers, optional Filter (nur aktive Sessions, nur getrennte Sessions, bestimmter Benutzer)."
---

# Remote Logged-On Users

## Zweck

Ermittelt, welche Benutzer auf einem entfernten Windows-System angemeldet sind — als Arbeitsauftrag, als Terminal-Server-Sitzung oder als RDP-Verbindung. Ergebnis sind die Sessions mit Benutzername, Session-ID, Zustand und Verbindungstyp.

Kernwerkzeug ist das Windows-Bordmittel `qwinsta` (Alias `query session`) bzw. `quser` (Alias `query user`). Beide nutzen RPC und funktionieren lokal wie remote über `/server:<Host>`. PowerShell-Alternativen decken Fälle ab, in denen RPC blockiert ist oder eine bestehende WinRM-Verbindung genutzt werden soll.

## Input

| Feld | Pflicht | Bedeutung |
|---|---|---|
| `host` | ja | Hostname oder IP des Ziel-Systems. Kein Pfad, kein URL-Schema. |
| `filter_state` | nein | `all` (Default), `active` (nur `STATE=Active`), `disconnected` (nur `STATE=Disc`) |
| `filter_user` | nein | Benutzername, auf den eingegrenzt wird (Teilstring, case-insensitiv) |
| `credentials` | nein | Nur wenn das Ziel andere Anmeldedaten verlangt. Niemals im Klartext in Argumente, Ausgaben oder Logs schreiben. |

## Output

Je Session eine Zeile:

| Feld | Herkunft | Bedeutung |
|---|---|---|
| `session_name` | `SESSIONNAME` | `console` (physische Konsole, ID 1), `rdp-tcp#N` (RDP), `services` (System) |
| `username` | `USERNAME` | Angemeldeter Benutzer; leer bei `services` |
| `session_id` | `ID` | Numerische Session-ID |
| `state` | `STATE` | `Active` = Client verbunden, `Disc` = getrennt, Session besteht weiter, `Down` / `Listen` = Sonderzustände |
| `type` | `TYPE` | `Console` oder `RDP` |
| `device` | `DEVICE` | Client-Name bei RDP-Verbindungen, sonst leer |

Zusätzlich: `logged_on_count` (Sessions mit belegtem `username`) und `active_count` (davon `state=Active`).

**Wichtig zur Auswertung:** `STATE=Active` bedeutet nur, dass der Client gerade verbunden ist. Ein getrennter User (`STATE=Disc`) ist weiterhin angemeldet — seine Prozesse laufen weiter, die Session wird beim Abmelden nicht beendet. Für „welche User sind angemeldet" zählen deshalb **alle Sessions mit belegtem `username`**, nicht nur die mit `STATE=Active`.

## Voraussetzungen

- `qwinsta` bzw. `quser` existiert nur auf Windows. Auf Linux und macOS gibt es kein RPC-basiertes Äquivalent.
- **RPC-Erreichbarkeit:** TCP 135 plus der dynamische Portbereich (üblicherweise TCP 49152–65535) muss zwischen Abfragesystem und Ziel firewallseitig freigegeben sein. Nur 135 freizugeben reicht nicht.
- **Berechtigungen:** Für die Remote-Abfrage sind Administratorrechte auf dem *Ziel* erforderlich. Ohne passende Rechte antwortet der RPC-Dienst des Ziels mit `Access is denied`.
- Für die PowerShell-Varianten mit `-ComputerName` wird **WinRM** (TCP 5985 bzw. 5986) und ein entsprechend konfigurierter Endpunkt benötigt.
- Die Spaltenüberschriften der Ausgabe sind sprachabhängig. Benutzernamen sind normalerweise nicht übersetzungsabhängig, Statuswerte (`Active`, `Disc`) können es sein.

## Vorgehen

1. **Ziel und Scope klären.** Frage nach, wenn Host oder Benutzerkreis unklar ist. Ein Terminal-Server hat typischerweise Dutzende Sessions — „angemeldet im Sinne laufender Prozesse" und „gerade aktiv am Client" sind verschiedene Fragen.
2. **Zuerst `quser` bzw. `qwinsta` probieren** — schnellster Weg, keine WinRM-Konfiguration nötig:

```powershell
& quser /server:'srv-host-01'
$exitCode = $LASTEXITCODE
```

3. **Ausgabe prüfen, nicht nur den Exit-Code.** `$LASTEXITCODE` ist 0, wenn der Befehl erfolgreich war — auch dann, wenn das Ergebnis fachlich leer ist (`No sessions exist`). Fehlertexte stehen auf stderr, die Session-Tabelle auf stdout. Beide getrennt erfassen.
4. **Filtern.** Die Ausgabe ist tabellarisch, nicht JSON. Zeilenweise in PowerShell verarbeiten; die Kopfzeile überspringen und Spalten anhand von Mehrfach-Leerzeichen trennen. Fertiges Skript in `references/command-reference.md`.
5. **Bei RPC-Fehlern umsteigen.** `Access is denied` → Berechtigungen auf dem *Ziel*, nicht die lokale Shell. `RPC server unavailable` → Firewall / dynamische Ports. Diagnosetabelle in `references/command-reference.md`.
6. **PowerShell-Alternative, wenn RPC gesperrt ist:**

```powershell
Get-CimInstance -ClassName Win32_ComputerSystem -ComputerName 'srv-host-01' |
    Select-Object -ExpandProperty UserName
```

> **Einschränkung:** `Win32_ComputerSystem.UserName` liefert **nur** den Benutzer der Konsolen-Session (ID 1). Auf einem Multi-User-Terminal-Server mit ausschließlich RDP-Sitzungen ist das Ergebnis typischerweise leer, obwohl viele User angemeldet sind. Für „alle angemeldeten User" ist `quser`/`qwinsta` die richtige Wahl.

## Fehlerdiagnose

| Meldung | Ursache | Handlung |
|---|---|---|
| `Access is denied.` | Fehlende Rechte auf dem Ziel für die RPC-Sitzungsabfrage | Auf dem Ziel als Administrator ausführen oder Ziel-Berechtigungen korrigieren; nicht die lokale Shell hochstufen |
| `RPC server unavailable.` | TCP 135 oder dynamischer RPC-Portbereich blockiert | Firewall des Ziels prüfen, Portbereich TCP 49152–65535 freigeben |
| `The network path was not found.` | Namensauflösung oder Routing zum Ziel | Hostname prüfen, IP direkt testen, DNS/Netzwerkpfad klären |
| `No sessions exist.` | Keine interaktive Session auf dem Ziel | Kein Fehler — Ergebnis ist fachlich leer; Rückgabe als „keine User angemeldet" formulieren |
| `quser` / `wuser.exe` nicht gefunden | Aufruf aus einer Umgebung ohne Systempfad | Über `C:\Windows\System32\quser.exe` aufrufen oder den Aufruf in `cmd /c` kapseln |

## Implementierung

Keine eigene Software nötig — der Skill nutzt ausschließlich Windows-Bordmittel. Reihenfolge der Präferenz:

1. `quser` (Alias `query user`) für den Normalfall: listet nur Sessions mit Benutzer, ohne die System-Session `services`.
2. `qwinsta` (Alias `query session`) wenn die vollständige Session-Liste inkl. `services` und Session-IDs gebraucht wird.
3. `Get-CimInstance -ClassName Win32_ComputerSystem -ComputerName <host>` als WinRM-Fallback — liefert nur den Konsolen-Benutzer, siehe Einschränkung oben.
4. `Get-CimInstance -ClassName Win32_Process -Filter "Name='explorer.exe'"` plus `Invoke-CimMethod -MethodName GetOwner` als weitere WinRM-Variante, wenn ein expliziter Explorer-Prozess je Session belegt werden soll.

Details, Skript-Vorlagen und die vollständige Befehlsreferenz stehen in `references/command-reference.md`.

## Quelle / Source

- **Befehlsreferenz:** Microsoft Learn — [qwinsta](https://learn.microsoft.com/windows-server/administration/windows-commands/qwinsta), [query user](https://learn.microsoft.com/windows-server/administration/windows-commands/query-user)
- **Pflege:** Dieser Skill wird im Repository [**BP-Agents**](https://github.com/schirkan/BP-Agents) gepflegt (Pfad: `.github/skills/remote-logged-on-users/`). Ergänzungen und Korrekturen bitte dort als PR einreichen.

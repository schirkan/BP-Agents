---
name: remote-session-logoff-reference
description: "Vollständige Befehls- und Skript-Referenz für den Skill remote-session-logoff: logoff, rwinsta und Win32_LogOff, Credential-Handhabung ohne Klartext-Passwörter, Verifikations-Skript, Auswirkungen der Abmeldung, Fehlermeldungen und Return-Codes."
---

# Remote Session Logoff — Befehls- und Skript-Referenz

Diese Referenz dokumentiert das tatsächliche Verhalten der Windows-Bordmittel `logoff`, `rwinsta` und der CIM-Klasse `Win32_LogOff`. Beispiele verwenden fiktive Hostnamen, Benutzernamen und Session-IDs.

## 1. Befehlsreferenz

### `logoff`

| Element | Bedeutung |
|---|---|
| `logoff <SessionID>` | Meldet die angegebene Session ab. |
| `/server:<Host>` | Führt den Vorgang auf dem entfernten System aus. Ohne den Schalter wirkt der Befehl lokal. |
| `/help` | Syntaxhilfe, Exit 0. |

Ohne Angabe einer Session-ID meldet `logoff` die **aktuelle** Session ab. Das ist in einer Remote-Kontext nicht gemeint und führt zum Abbruch der eigenen Arbeitssitzung.

Exit-Code: `0` bei Erfolg, `1` bei Fehler. Es gibt keine gesonderte Rückgabe, die Erfolg von „Session war schon weg" unterscheidet — daher ist die Nachkontrolle zwingend.

### `rwinsta`

| Element | Bedeutung |
|---|---|
| `rwinsta <SessionID>` | Setzt die angegebene Session zurück (Reset). |
| `/server:<Host>` | Führt den Vorgang auf dem entfernten System aus. |
| `/help` | Syntaxhilfe, Exit 0. |

Exit-Code: `0` bei Erfolg, `1` bei Fehler.

### `query user` / `quser`

Nur lesend, liefert die Session-Liste inklusive Session-ID. Details im Skill `remote-logged-on-users`.

## 2. `logoff` und `rwinsta` im Vergleich

| Aspekt | `logoff` | `rwinsta` |
|---|---|---|
| Ablauf | Orderly Shutdown | Harter Reset |
| Benachrichtigung an Anwendungen | Ja — Anwendungen können noch speichern und sich sauber beenden | Nein — Abbruch ohne Vorwarnung |
| Datenverlustrisiko | Gering, aber nicht null | Erhöht, ungespeicherte Arbeit geht verloren |
| Auswirkung auf laufende Prozesse | Prozesse werden beendet, Session-ID wird freigegeben | Prozesse werden sofort beendet, Session wird zurückgesetzt |
| Empfehlung | **Standard** | Nur bei ausdrücklicher Weisung, z. B. wenn eine Session hängt und orderly nicht greift |

## 3. Credential-Handhabung

Der Auftrag setzt ein Passwort voraus. Die Bordmittel `logoff` und `rwinsta` kennen **keine eigene Credential-Option** — sie verwenden den Windows-Anmeldetoken des aktuellen Kontexts. Es gibt drei Wege, das Passwort bereitzustellen.

### 3.1 Empfohlen: `Invoke-Command` über WinRM

Der `logoff` läuft dadurch **auf dem Ziel selbst**, lokal und mit Administratorrechten. Es gibt keine RPC-Authentifizisierung über die Leitung, und das Passwort erscheint nirgends in Argumenten.

```powershell
$cred = Get-Credential
Invoke-Command -ComputerName 'srv-host-01' -Credential $cred -ScriptBlock {
    logoff 2
    $LASTEXITCODE
}
```

Voraussetzung: WinRM auf dem Ziel aktiv (TCP 5985 bzw. 5986), PowerShell-Remoting eingerichtet.

### 3.2 `Win32_LogOff` über CIM

```powershell
$cred = Get-Credential
$result = Invoke-CimMethod -ClassName Win32_LogOff -MethodName LogOff \
    -Arguments @{ SessionId = 2 } \
    -ComputerName 'srv-host-01' -Credential $cred

$result.ReturnValue
```

Signatur der Methode: `uint32 LogOff(uint32 SessionId, uint32 Reason)`. `SessionId` bezeichnet die Ziel-Session, `Reason` den Abbruchgrund.

### 3.3 Credential-Manager und `net use`

Wenn der Aufruf zwingend über das Bordmittel `logoff` mit `/server:` laufen muss, muss die Verbindung zum Ziel vorher authentifiziert sein.

```powershell
# Passwort interaktiv hinterlegen, ohne es in die Kommandozeile zu schreiben
cmdkey /add:srv-host-01 /user:DOMAIN\admin
```

`cmdkey` fragt das Passwort interaktiv ab und legt es im Windows Credential Manager ab. Danach funktioniert `logoff /server:srv-host-01` mit den hinterlegten Daten.

Alternative mit `net use` — **mit Passwort im Klartext in der Kommandozeile**, sichtbar in der Prozessliste und in der PowerShell-History:

```powershell
net use \\srv-host-01\IPC$ /user:DOMAIN\admin PASSWORD
```

> **Diese Variante nur in kontrollierten Umgebungen verwenden.** Das Passwort ist über `Get-Process`, Event-Logs und die Shell-History einsehbar. Wenn möglich `cmdkey` ohne Passwortargument oder den WinRM-Weg nutzen.

### Übersicht

| Variante | Passwort im Klartext in Argumenten? | Benötigt WinRM? | RPC über die Leitung? |
|---|---|---|---|
| `Invoke-Command` + `logoff` | nein | ja | nein (läuft lokal auf dem Ziel) |
| `Win32_LogOff` via CIM | nein | ja | nein (läuft über WinRM) |
| `cmdkey` + `logoff /server:` | nein | nein | ja |
| `net use` + `logoff /server:` | **ja** | nein | ja |

## 4. Verifikations- und Ausführungsskript

Das Skript führt keine Aktion ohne explizite Bestätigung aus. `-Confirm` ist bewusst als interaktiver Schalter eingebaut.

```powershell
function Invoke-RemoteSessionLogoff {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)] [string] $ComputerName,
        [Parameter(Mandatory)] [int]    $SessionId,
        [Parameter(Mandatory)] [string] $ExpectedUser,
        [ValidateSet('logoff', 'rwinsta')] [string] $Mode = 'logoff'
    )

    # 1. Session verifizieren
    $raw = & quser "/server:$ComputerName" 2>&1
    if ($LASTEXITCODE -ne 0) {
        Write-Error "quser fehlgeschlagen (Exit $LASTEXITCODE): $raw"
        return
    }

    $sessions = $raw | Select-Object -Skip 1 | ForEach-Object {
        $line = [string]$_
        if ($line -notmatch '^\s*(?<session>\S+)\s{2,}(?<user>\S*)\s{2,}(?<id>\d+)\s{2,}(?<state>\S+)') { return }
        [pscustomobject]@{
            session_name = $Matches.session
            username    = ($Matches.user -replace '^>', '')
            session_id  = [int]$Matches.id
            state       = $Matches.state
        }
    }

    $target = $sessions | Where-Object { $_.session_id -eq $SessionId }
    if (-not $target) {
        Write-Output "status=not_found host=$ComputerName session_id=$SessionId"
        return
    }

    # 2. Benutzername abgleichen
    if ($target.username -ne $ExpectedUser) {
        Write-Output "status=mismatch host=$ComputerName expected=$ExpectedUser actual=$($target.username)"
        $sessions | Format-Table | Out-String | Write-Output
        return
    }

    # 3. Bestätigung einholen
    $summary = "Sitzung $SessionId ($($target.session_name)) von '$($target.username)' auf $ComputerName abmelden [STATE=$($target.state)]"
    if (-not $PSCmdlet.ShouldProcess($summary, "Session-Abmeldung ($Mode)")) {
        Write-Output "status=declined host=$ComputerName session_id=$SessionId"
        return
    }

    # 4. Ausführen
    $cred = Get-Credential
    $exit = Invoke-Command -ComputerName $ComputerName -Credential $cred -ScriptBlock {
        param($Id, $Cmd)
        & $Cmd $Id
        $LASTEXITCODE
    } -ArgumentList $SessionId, $Mode

    # 5. Nachkontrollieren
    $after = & quser "/server:$ComputerName" 2>&1
    $stillThere = ($after | Select-Object -Skip 1 | Where-Object { [string]$_ -match "\s$SessionId\s" })

    if ($stillThere) {
        Write-Output "status=verification_failed host=$ComputerName session_id=$SessionId exit=$exit"
    } else {
        Write-Output "status=success host=$ComputerName session_id=$SessionId user=$($target.username) mode=$Mode exit=$exit"
    }
}
```

Aufruf:

```powershell
Invoke-RemoteSessionLogoff -ComputerName 'srv-host-01' -SessionId 2 -ExpectedUser 'JohnDoe' -Mode logoff -Confirm
```

**Vor dem Ausführen prüfen, was in der Session läuft.** Bei einem RPA-Client oder einem Editor kann die Abmeldung laufende Arbeit zerstören:

```powershell
Invoke-Command -ComputerName 'srv-host-01' -Credential $cred -ScriptBlock {
    Get-Process | Where-Object { $_.SessionId -eq 2 } |
        Select-Object Name, SessionId, StartTime
}
```

## 5. Auswirkungen der Abmeldung

- Alle Prozesse der Session werden beendet. Bei `logoff` orderly, bei `rwinsta` hart.
- Offene RDP-Verbindungen zur Session brechen ab.
- Die Session-ID wird freigegeben und bei der nächsten Anmeldung neu vergeben. Gespeicherte Session-IDs sind danach wertlos.
- Ungespeicherte Arbeit in Anwendungen geht verloren. Bei `logoff` können Anwendungen auf `WM_QUERYENDSESSION` noch reagieren, das ist aber nicht garantiert.
- **RPA-Hinweis:** Läuft in der Session ein Blue-Prism- oder anderer Automatisierungsclient, bricht dieser ohne Vorwarnung ab. Work-Queue-Items können in einem Zwischenstatus hängen bleiben. Vor der Abmeldung prüfen, ob ein Prozess aktiv ist, und den Nutzer darauf hinweisen.

## 6. Fehlermeldungen und Return-Codes

### `logoff` / `rwinsta`

| Meldung | Exit | Ursache | Handlung |
|---|---|---|---|
| `ERROR: Access is denied.` | 1 | Keine Administratorrechte auf dem Ziel oder Credentials fehlen/falsch | Credentials prüfen, Rechte auf dem Ziel bestätigen |
| `ERROR: The system could not find the session specified.` | 1 | Session-ID existiert nicht oder ist veraltet | `quser /server:<host>` neu ausführen |
| `ERROR: RPC server unavailable.` | 1 | TCP 135 oder dynamischer RPC-Portbereich blockiert | Firewall prüfen |
| `ERROR: The network path was not found.` | 1 | Namensauflösung oder Routing fehlgeschlagen | Hostname und IP prüfen |
| `No session exists.` | 0 | Keine abmeldbare Session vorhanden | Kein Fehler, aber kein Eingriff — Verifikation prüfen |

### `Win32_LogOff.ReturnValue`

| Wert | Bedeutung | Handlung |
|---|---|---|
| 0 | Erfolg | Nachkontrolle fahren |
| 2 | Objekt / Session nicht gefunden | Session-Liste neu abfragen, ID korrigieren |
| 5 | Zugriff verweigert | Credentials und Rechte auf dem Ziel prüfen |
| 8 | Nicht unterstützt | Methode steht auf dem Ziel nicht zur Verfügung — auf `logoff` ausweichen |
| 9 | Allgemeiner Fehler | Meldung im CIM-Fehlerobjekt auswerten |

## 7. Quellen

- Microsoft Learn — [logoff](https://learn.microsoft.com/windows-server/administration/windows-commands/logoff)
- Microsoft Learn — [rwinsta](https://learn.microsoft.com/windows-server/administration/windows-commands/rwinsta)
- Microsoft Learn — [query user](https://learn.microsoft.com/windows-server/administration/windows-commands/query-user)
- Microsoft Learn — [Win32_LogOff-Klasse](https://learn.microsoft.com/windows/win32/wmistrng/wmistrng-base/win32-logoff)
- Microsoft Learn — [cmdkey](https://learn.microsoft.com/windows-server/administration/windows-commands/cmdkey)

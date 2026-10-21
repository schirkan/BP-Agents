---
name: remote-logged-on-users-reference
description: "Vollständige Befehls- und Ausgabe-Referenz für den Skill remote-logged-on-users: Ausgabeformate von quser und qwinsta, robuste PowerShell-Auswertung, CIM-/WinRM-Alternativen, Netzwerk- und Firewall-Anforderungen, Fehlermeldungen und Exit-Codes."
---

# Remote Logged-On Users — Befehls- und Ausgabe-Referenz

Diese Referenz dokumentiert das tatsächliche Verhalten der Windows-Bordmittel `quser`, `qwinsta` und der CIM-Varianten. Beispiele verwenden fiktive Hostnamen, Benutzernamen und Session-IDs.

## 1. Ausgabeformat

### `quser` (Alias `query user`)

```
 USERNAME       SESSIONNAME     ID  STATE    TYPE        DEVICE
>JohnDoe        console         1  Active
 JaneSmith      rdp-tcp#5       2  Disc
>Admin          rdp-tcp#8       3  Active   RDP         CLIENT-PC-07
```

- `>` kennzeichnet die Session, in der der Aufruf selbst läuft. Das Zeichen gehört zum Benutzernamen und muss vor dem Filtern entfernt werden.
- Die System-Session `services` erscheint **nicht**.
- Spaltenreihenfolge: `USERNAME`, `SESSIONNAME`, `ID`, `STATE`, `TYPE`, `DEVICE`.

### `qwinsta` (Alias `query session`)

```
 SESSIONNAME       USERNAME        ID  STATE    TYPE        DEVICE
 services                          0  Disc
 console           JohnDoe         1  Active
 rdp-tcp#5         JaneSmith       2  Disc
```

- Die System-Session `services` (ID 0) erscheint mit **leerem** Benutzernamen.
- Spaltenreihenfolge: `SESSIONNAME`, `USERNAME`, `ID`, `STATE`, `TYPE`, `DEVICE`.

### Zustände (`STATE`)

| Wert | Bedeutung |
|---|---|
| `Active` | Der Client ist gerade verbunden und interaktiv angemeldet. |
| `Disc` | Der Client ist getrennt. Die Session besteht, die Prozesse laufen weiter. Der User ist weiterhin angemeldet. |
| `Down` | Die Session ist heruntergefahren, die Daten werden gerade bereinigt. |
| `Listen` | Die Session wartet auf eine Verbindungsübernahme. |

**Die häufigste Fehldeutung:** `Active` ist nicht gleich „angemeldet". Ein getrennter User ist angemeldet, seine Automationen laufen. Für „welche User sind gerade auf dem System" gehören `Active` **und** `Disc` zur Antwort.

## 2. Auswertung in PowerShell

Die Ausgabe ist tabellarisch, nicht JSON. Einfaches Splitten an einem Trennzeichen scheitert an der leeren Benutzerspalte der `services`-Zeile. Der folgende Ansatz nutzt eine Positionsregularausdruck und ist für beide Befehle stabil.

```powershell
function Get-RemoteSession {
    param(
        [Parameter(Mandatory)] [string] $ComputerName,
        [ValidateSet('quser', 'qwinsta')] [string] $Command = 'quser'
    )

    $raw = & $Command "/server:$ComputerName" 2>&1
    $exitCode = $LASTEXITCODE

    if ($exitCode -ne 0) {
        Write-Error "Abfrage fehlgeschlagen (Exit $exitCode): $raw"
        return
    }

    $rows = $raw | Select-Object -Skip 1 | ForEach-Object {
        $line = [string]$_
        if ($line -notmatch '^\s*(?<session>\S+)\s{2,}(?<user>\S*)\s{2,}(?<id>\d+)\s{2,}(?<state>\S+)') {
            return
        }
        [pscustomobject]@{
            session_name = $Matches.session
            username    = $Matches.user -replace '^>', ''
            session_id  = [int]$Matches.id
            state       = $Matches.state
        }
    }

    $rows | Where-Object { $_.username }
}
```

Aufruf und Filter:

```powershell
$sessions = Get-RemoteSession -ComputerName 'srv-host-01'

# nur User, die wirklich angemeldet sind (ohne System-Session)
$loggedOn = $sessions | Where-Object { $_.username -and $_.username -ne 'services' }

# gerade aktiv am Client
$active   = $loggedOn | Where-Object { $_.state -eq 'Active' }

# getrennt, aber Prozesse laufen weiter
$disconnected = $loggedOn | Where-Object { $_.state -eq 'Disc' }

# bestimmter Benutzer
$one = $loggedOn | Where-Object { $_.username -like '*Smith*' }

[pscustomobject]@{
    logged_on_count   = @($loggedOn).Count
    active_count      = @($active).Count
    disconnected_count = @($disconnected).Count
} | Format-List
```

**Zu beachten:**

- `$LASTEXITCODE` direkt nach dem Aufruf erfassen, bevor weitere native Befehle laufen.
- `2>&1` vermischt Fehlermeldungen mit der Tabelle. Für maschinelle Auswertung stderr getrennt halten, sonst landen Fehlerzeilen in der Parser-Logik.
- Die Regex erwartet mindestens zwei Leerzeichen zwischen den Spalten. Auf Systemen mit abweichender Formatierung kann die Spaltentrennung einzelner Tabulator sein; dann `\s{2,}` zu `[\s\t]{2,}` erweitern.
- `USERNAME`-Werte können domainspezifisch präfixiert sein (z. B. `DOMAIN\\JohnDoe`). Beim Filtern auf den reinen Namen die Domain mit abtrennen.

## 3. Befehlsreferenz

### `quser` / `query user`

| Element | Bedeutung |
|---|---|
| `/server:<Host>` | Abfrage auf dem entfernten System. Ohne den Schalter wird lokal abgefragt. |
| `/help` | Syntaxhilfe, Exit 0. |

Listet ausschließlich Sessions mit angemeldetem Benutzer. Für die Frage „welche User sind angemeldet" die erste Wahl, weil die leere System-Session-Zeile entfällt und das Parsing einfacher wird.

### `qwinsta` / `query session`

| Element | Bedeutung |
|---|---|
| `/server:<Host>` | Abfrage auf dem entfernten System. |
| `/help` | Syntaxhilfe, Exit 0. |

Listet alle Sessions inkl. `services` und inkl. `ID`. Nötig, wenn die vollständige Session-Landkarte gebraucht wird oder eine Session-ID aus der Ausgabe in Folgeabfragen übernommen werden soll.

### Gemeinsam

- Beide Befehle nutzen RPC (MS-RPCE) und benötigen den RPC-Server auf dem Ziel.
- Kein eigener Output-Parameter, kein JSON-Format. Die Ausgabe ist immer die formatierte Tabelle.
- Exit-Code 0 bei erfolgreicher Ausführung, auch bei leerem Ergebnis.

## 4. PowerShell-Alternativen (WinRM)

Setzen eine erreichbare WinRM-Verbindung voraus (TCP 5985 oder 5986, aktivierter WinRM-Dienst, TrustedHosts-Konfiguration bei IP-Adressen).

### 4.1 Konsolen-Benutzer

```powershell
Get-CimInstance -ClassName Win32_ComputerSystem -ComputerName 'srv-host-01' |
    Select-Object -ExpandProperty UserName
```

**Grenze:** liefert nur den Benutzer der **Konsolen-Session** (ID 1). Auf einem Server, auf dem niemand an der physischen Konsole arbeitet, ist das Ergebnis leer. `Get-WmiObject` ist deprecated und in PowerShell 7 nicht mehr enthalten — `Get-CimInstance` verwenden.

### 4.2 Explorer-Prozess je Session

```powershell
$procs = Get-CimInstance -ClassName Win32_Process -Filter "Name='explorer.exe'" -ComputerName 'srv-host-01'

foreach ($p in $procs) {
    $owner = Invoke-CimMethod -InputObject $p -MethodName GetOwner
    [pscustomobject]@{
        username    = $owner.User
        domain      = $owner.Domain
        session_id  = $p.SessionId
        process_id  = $p.ProcessId
    }
}
```

**Grenzen:** ein Explorer-Prozess je interaktiver Session ist ein Indiz, kein Beweis. Dienstkonten ohne Explorer, gesperrte Sessions oder mehrere Explorer-Prozesse je Session verfälschen das Ergebnis. `Invoke-CimMethod` mit `GetOwner` benötigt auf dem Ziel administrative Rechte für fremde Prozesse.

### 4.3 Lokale Prozess-Sicht

```powershell
Get-Process -Name explorer -IncludeUserName -ErrorAction SilentlyContinue |
    Select-Object UserName, SessionId
```

**Grenze:** `-IncludeUserName` wirkt nur **lokal**. Fremde Sessions sind nur mit lokalen Administratorrechten sichtbar, und `Get-Process -ComputerName` unterstützt die Kombination mit `-IncludeUserName` nicht.

### Methodenvergleich

| Methode | Alle User | Konsolen-User | Getrennte | Transport | Lokale Admin nötig |
|---|---|---|---|---|---|
| `quser /server:<host>` | ja | ja | ja | RPC | nein |
| `qwinsta /server:<host>` | ja | ja | ja | RPC | nein |
| `Win32_ComputerSystem.UserName` | **nein** | ja | nein | WinRM | nein |
| `Win32_Process` + `GetOwner` | ja | ja | ja | WinRM | nein (Zielrechte) |
| `Get-Process -IncludeUserName` | ja | ja | ja | lokal | **ja** |

## 5. Netzwerk und Firewall

| Port | Zweck | Blockade-Folge |
|---|---|---|
| TCP 135 | RPC-Endpunkt-Mapper | `RPC server unavailable` |
| TCP 49152–65535 | Dynamischer RPC-Portbereich | Verbindung bricht nach dem Mapper-Kontakt ab |
| TCP 5985 / 5986 | WinRM (nur PowerShell-Varianten) | CIM-Abfragen schlagen fehl |

Freigabe nur auf Port 135 reicht nicht — der RPC-Dienst weist den konkreten Port pro Verbindung dynamisch zu. Alternativ lässt sich der Portbereich per Registry auf dem Ziel einschränken.

Schnelltest der Erreichbarkeit:

```powershell
Test-NetConnection -ComputerName 'srv-host-01' -Port 135
```

## 6. Fehlermeldungen und Exit-Codes

| Meldung | Exit | Ursache | Handlung |
|---|---|---|---|
| `Access is denied.` | 1 | RPC-Autorisierung auf dem Ziel lehnt die Abfrage ab | Auf dem Ziel administrative Rechte sicherstellen; die Rechte des aufrufenden Kontos auf dem Ziel prüfen |
| `RPC server unavailable.` | 1 | TCP 135 oder dynamischer Portbereich blockiert | Firewall des Ziels prüfen, Portbereich freigeben |
| `The network path was not found.` | 1 | Namensauflösung oder Routing fehlgeschlagen | Hostname und IP-Adresse testen, DNS- und Netzwerkpfad klären |
| `The specified network name is no longer available.` | 1 | Ziel nicht erreichbar oder RPC-Dienst beendet | Erreichbarkeit prüfen, Firewall-Regeln kontrollieren |
| `No sessions exist.` | 0 | Keine interaktive Session vorhanden | Kein Fehler. Ergebnis als „keine User angemeldet" formulieren |
| `quser` nicht gefunden | — | Aufruf ohne Systempfad im `PATH` | Über `C:\Windows\System32\quser.exe` aufrufen oder den Aufruf in `cmd /c` kapseln |

Ein Exit-Code von 0 belegt nur, dass der Befehl ausgeführt werden konnte — nicht, dass Sessions vorhanden sind. Ausgabe und Exit-Code immer gemeinsam auswerten.

## 7. Quellen

- Microsoft Learn — [qwinsta](https://learn.microsoft.com/windows-server/administration/windows-commands/qwinsta)
- Microsoft Learn — [query user](https://learn.microsoft.com/windows-server/administration/windows-commands/query-user)
- Microsoft Learn — [Get-CimInstance](https://learn.microsoft.com/powershell/module/cimcmdlets/get-ciminstance)
- Microsoft Learn — [Get-Process](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-process)

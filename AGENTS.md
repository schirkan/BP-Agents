# BP-Agents

Sammlung von KI-Agenten und Skill-Dateien. Dieses Repo ist so strukturiert, dass **GitHub Copilot** (Coding Agent, Chat, Workspace) die zentralen Anweisungen aus dieser Datei laden kann.

## Zweck

- `.github/agents/` enthält **Agent-Definitionen** (eine Datei pro Agent).
- `.github/skills/` enthält **Skill-Definitionen** (eine Datei pro Skill, wiederverwendbar über mehrere Agents hinweg).

Diese beiden Ordner folgen der GitHub-Copilot-Konvention und werden vom Copilot Coding Agent automatisch erkannt.

## Konventionen

- Pro Agent **eine Datei** in `.github/agents/`, pro Skill **eine Datei** in `.github/skills/`.
- Dateinamen sind **kebab-case** und beschreibend, z. B. `.github/agents/incident-triage.md`, `.github/skills/http-fetch.md`.
- Format bevorzugt **Markdown** (für Lesbarkeit in PR-Reviews und Copilot-Kontext). JSON/YAML erlaubt, wenn maschinenlesbar nötig.
- Quellcode, Daten und Konfiguration, die ein Agent oder Skill braucht, liegen **im selben Unterordner** wie die jeweilige Definitionsdatei.
- Keine Geheimnisse, Tokens oder personenbezogene Daten einchecken.

## Agent-Schema (`.github/agents/*.md`)

Jede Agent-Datei enthält diese Sektionen (in dieser Reihenfolge):

| Sektion | Pflicht? | Zweck |
|---|---|---|
| `## Name` | ja | Eindeutiger Name des Agents |
| `## Zweck` | ja | Was tut der Agent, in ein bis zwei Sätzen |
| `## Trigger` | ja | Wann / wie wird er aktiv (manuell, Event, Schedule, Chat-Befehl) |
| `## Skills` | ja | Liste der Skills, die der Agent nutzt (Dateinamen aus `.github/skills/`) |
| `## Tools` | nein | Externe Systeme, APIs, MCP-Server, die der Agent aufruft |
| `## System-Prompt` | ja | Verhaltensregeln, Ton, Grenzen, Eskalation |
| `## Beispiele` | nein | Beispiel-Inputs/-Outputs zur Veranschaulichung |

## Skill-Schema (`.github/skills/*.md`)

Jede Skill-Datei enthält diese Sektionen:

| Sektion | Pflicht? | Zweck |
|---|---|---|
| `## Name` | ja | Eindeutiger Name des Skills |
| `## Zweck` | ja | Was kapselt der Skill |
| `## Input` | ja | Erwartete Eingaben (Schema, Typen, Beispiel) |
| `## Output` | ja | Rückgabewert (Schema, Typen, Beispiel) |
| `## Voraussetzungen` | nein | Externe Abhängigkeiten, Credentials (Namen, nie Werte), Berechtigungen |
| `## Implementierung` | nein | Verweis auf Code, Skript oder Pseudocode, der den Skill ausführt |

## Ergänzen eines Agents

1. Datei `.github/agents/<name>.md` anlegen, Schema oben befolgen.
2. Benötigte Skills in `## Skills` referenzieren (sie müssen in `.github/skills/` existieren).
3. PR öffnen, Reviewer:in markieren, kurze Beschreibung im PR-Body.

## Ergänzen eines Skills

1. Datei `.github/skills/<name>.md` anlegen, Schema oben befolgen.
2. `## Implementierung` so konkret wie möglich (Pfad zu Skript, Funktionssignatur, oder kurzer Pseudocode).
3. Wenn Credentials nötig sind: nur Namen referenzieren, Werte über Secret-Management (z. B. GitHub Actions Secrets, `.env` lokal — siehe `.gitignore`).

## Lizenz

[MIT](./LICENSE)

## Git

- **Repo-Typ:** GitHub (public)
- **Pfad / URL:** `https://github.com/schirkan/BP-Agents`
- **Remote(s):** `origin = https://github.com/schirkan/BP-Agents.git`
- **Eingerichtet am:** 2026-10-16
- **Standard-Branch:** `main`
- **`.gitignore`-Status:** vorhanden
- **Initial-Commit:** `bfdf363`

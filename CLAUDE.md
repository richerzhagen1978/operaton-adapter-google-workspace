# CLAUDE.md – operaton-adapter-google-workspace

Diese Datei wird von Claude Code beim Start automatisch geladen.  
Vollständiger Projektkontext und Chatverlauf: `open-items.md` und `chat_history.md`.

---

## Projekt auf einen Blick

| | |
|---|---|
| **Ziel** | Modularer Operaton-Adapter für Google Workspace (Gmail, Calendar, …) |
| **Betreiber** | Björn Richerzhagen, MINAUTICS GmbH, Berlin |
| **Repo** | https://github.com/richerzhagen1978/operaton-adapter-google-workspace |
| **Freigaben** | Björn entscheidet alle Merges nach `main` und Prod-Deployments |

---

## Architektur (beschlossen)

```
Operaton Engine
      │  External Task API (fetchAndLock / complete / failure)
      ▼
┌─────────────────────────────────────┐
│         Google-Workspace-Adapter    │
│                                     │
│  Core-Modul                         │
│  ├── Auth (Domain-Wide Delegation)  │
│  ├── Token-Refresh                  │
│  ├── Retry / Backoff                │
│  ├── Konfiguration & Allowlist      │
│  └── Logging                        │
│                                     │
│  Service-Module (nur aktivierte)    │
│  ├── Gmail    (MVP)                 │
│  ├── Calendar (MVP)                 │
│  └── Drive / Sheets / … (später)   │
└─────────────────────────────────────┘
      │  googleapis (Service Account)
      ▼
Google Workspace API
```

**Kanal-Prinzip:** Operaton und Google kennen sich nicht. Der Adapter kanalisiert alles.

**Auth:** Domain-Wide Delegation via Service Account.  
- Jede Operation hat ein `impersonateAs`-Feld (Default = Funktionspostfach, Override pro Task).  
- Allowlist der erlaubten Postfächer/Nutzer liegt in der Adapter-Konfiguration.  
- Service-Account-Key **niemals** im Repo oder in Logs – ausschließlich über Secret-Store / Umgebungsvariablen.

**Anwendungsfall (noch offen):**  
- Outbound (Mail senden, Termin anlegen) → empfohlener MVP-Start  
- Inbound-Trigger (Cloud Pub/Sub / Watch-Channels) → separates Modul, später

---

## Pflichtregeln – immer einhalten

### A1 · Release-Workflow

```
Feature-Branch → develop (CI grün) → Testanweisung an Björn → Freigabe → main
```

1. Arbeit auf Feature-Branch, Merge nach `develop`.
2. Stage-CI muss grün sein.
3. Claude liefert Testanweisungen für Björn.
4. Merge nach `main` und Prod-Deployment **nur mit Björns ausdrücklicher Freigabe**.
5. Kein Direkt-Push auf `main`. Keine Ausnahmen, auch nicht für kleine Fixes.

### A2 · Hotfix-Pfad

Hotfix-Branch → CI grün → minimale Testanweisung → Freigabe Björn → `develop`.  
Was als Hotfix gilt, entscheidet Björn.

### A3 · Wiederverwendung

Vor neuem Code prüfen, ob vorhandene Bausteine passen.  
Muster wird ausgelagert, sobald es 2–3 Mal vorkommt. Keine vorschnelle Abstraktion.

### A4 · Business-Logik im Backend

Fachlogik liegt im Backend. Abweichung → vorher Rücksprache.

### A5 · Dokumentation

- Architekturentscheidungen in `DECISIONS.md` (versioniert im Repo), nicht nur im Memory.
- Zugänge: nur festhalten, wo und wie sie geregelt sind. Keine Passwörter, Keys oder Tokens im Repo.

---

## Sicherheit (immer)

### B1 · Mandanten & Berechtigungen

- Jede Backend-Route prüft Nutzer → Mandant → Berechtigung.
- Mandantenfilter in **jede** Datenbankabfrage, nicht nur in die Route.
- Testpflicht: „fremder Mandant wird abgewiesen".

### B2 · Permission-Regel

| Kontext | Funktion |
|---------|---------|
| Backend (fachlich) | `requirePerm('schluessel')` |
| Frontend | `window.hasPerm('schluessel')` |
| Backend (technisch-admin) | `requireRole` – **nur** für Nutzerverwaltung, Berechtigungsmatrix, System-Config |

Kein `me.role === …` im Frontend zum Ein-/Ausblenden von UI.  
Ausnahme: negative Abgrenzung von Gast-/Kundenrollen (`me.role === 'gast'`).

### B3 · Reihenfolge bei neuen Features (pro PR, nicht pro Commit)

1. Migration  
2. `PERM_KEYS`  
3. `PERM_LABELS`  
4. `PERM_HELP`  
5. Route  
6. Frontend  
7. Tests  

PR wird nicht gemergt, bevor alle 7 Schritte enthalten sind.  
Vor dem Merge: `requireRole` in geänderten Route-Dateien → begründen oder durch `requirePerm` ersetzen.

### B4 · Geheimnisse

Keys, Tokens, Passwörter → **ausschließlich Secret-Verwaltung der Umgebung**.  
Niemals in Code, Logs, Memory oder Testdaten.

---

## Öffentliche API (nur wenn externe Nutzer)

- Innerhalb einer Version nur **additive** Änderungen (neue optionale Felder, neue Endpunkte).
- Breaking Changes (Feld entfernen, Typ ändern, Pflichtfeld ergänzen, Statuscode ändern) → neue Version.
- Kein API-Merge ohne aktualisierte OpenAPI-Spezifikation.
- Tests: immer Erfolgsfall **und** Ablehnungsfälle (falscher Key, fehlender Scope, fremder Mandant, doppelte `external_id`, Ratenlimit).
- Fehlerformat: `{ code, message, request_id }`.

---

## Frontend / UI (nur wenn eigene UI vorhanden)

- **Tokens statt Magic Numbers:** Farben, Größen, Abstände aus CSS-Variablen. Raster: 4/8/12/16/24/32 px.
- **Semantische Struktur:** Genau ein `<h1>`, lückenlose Hierarchie, Landmarks (`<header>`, `<nav>`, `<main>`).
- **WCAG 2.1 AA:** Kontrast Text ≥ 4,5:1 / UI-Elemente ≥ 3:1; sichtbarer `:focus-visible`; Modals mit Fokus-Trap; Touch-Ziele ≥ 44 × 44 px.
- **Responsive:** Mobile-first, Breakpoints 640/1024/1280 px.
- **i18n:** Keine hartkodierten UI-Strings. Alles über Übersetzungsschlüssel, inkl. `aria-label`, Tooltips, Leerzustände. Datums-/Zahlenformate via Intl-API.

---

## CI-Checks (automatisiert)

| Check | Werkzeug |
|-------|---------|
| Kein `requireRole` in Route-Dateien ohne Begründung | Lint-Skript |
| Kein Direkt-Push auf `main` | Branch Protection |
| Keine Magic Numbers / fremde Farben | Stylelint |
| Keine hartkodierten UI-Strings | i18n-Lint |
| Barrierefreiheit | axe |
| OpenAPI aktuell, Verträge eingehalten | OpenAPI-Lint / Contract-Tests |
| Keine Geheimnisse im Repo | Secret-Scanning |

---

## Offene Punkte (Stand 2026-10-03)

| # | Bereich | Was fehlt |
|---|---------|-----------|
| 1 | Architektur | CIP-Adapter-Code bereitstellen (Vorlage für diesen Adapter) |
| 2 | Auth | Service Account anlegen + Domain-Wide Delegation konfigurieren |
| 3 | Auth | Secret-Store für Service-Account-Key festlegen |
| 4 | Core | Allowlist-Konzept definieren |
| 5 | Anwendungsfall | Entscheiden: Outbound-only MVP oder auch Inbound-Trigger |
| 6 | Repo | Branch `develop` anlegen, Branch Protection auf `main` setzen |
| 7 | Repo | CI-Pipeline aufsetzen (Lint, Tests, Secret-Scanning) |
| 8 | Implementierung | Core-Modul (Auth, Retry/Backoff, Konfiguration, Logging) |
| 9 | Implementierung | Service-Modul Gmail – Mail senden (MVP) |
| 10 | Implementierung | Service-Modul Calendar – Termin anlegen (MVP) |
| 11 | Später | Trigger-Modul Gmail (Cloud Pub/Sub oder Polling) |
| 12 | Später | Trigger-Modul Calendar (Watch-Channels oder Polling) |

Vollständiger Kontext: → `open-items.md`  
Bisheriger Chatverlauf: → `chat_history.md`

# Open Items & Projektkontext – operaton-adapter-google-workspace

Diese Datei dient Claude Code als vollständige Arbeitsgrundlage. Sie enthält alle gespeicherten Informationen über Projektkontext, Projektrichtlinien und offene Punkte.

**Primäre Einstiegsdatei für Claude Code:** `CLAUDE.md`  
**Vollständiger Chat-Verlauf:** `chat_history.md`

Zuletzt aktualisiert: 2026-10-03

---

## 1. Projektkontext

**Ziel:** Adapter für die BPMN-Prozessengine [Operaton](https://operaton.org), der Google-Workspace-Dienste anbindet – zunächst Gmail und Calendar, später Drive, Sheets u. a.

**Repo:** https://github.com/richerzhagen1978/operaton-adapter-google-workspace

**Betreiber:** Björn Richerzhagen, MINAUTICS GmbH, Berlin

### Kernarchitektur (beschlossen)
- **Modularer Google-Workspace-Adapter** – kein Gmail-only-Adapter, kein separater Adapter je Produkt.
- **Core-Modul:** Auth (Domain-Wide Delegation via Service Account), Token-Refresh, Retry/Backoff bei Quotas, zentrale Konfiguration, Logging.
- **Service-Module:** Gmail, Calendar, Drive, Sheets usw. – je eigene Operationen und eigene Scopes, nur die konfigurierten Module werden geladen.
- **MVP-Scope:** Core + Gmail + Calendar.

### Architekturgrundsatz (Kanal-Prinzip)
Operaton und Google Workspace kennen sich nicht – der Adapter kanalisiert alles:
- **Operaton-Seite:** Adapter spricht Operaton External Task API (Poll/FetchAndLock/Complete/Failure)
- **Google-Seite:** Adapter spricht Google-APIs (googleapis)

### Auth-Konzept (beschlossen)
- **Domain-Wide Delegation** mit einem Google Service Account.
- Ermöglicht Impersonation von Domain-Nutzern ohne interaktive OAuth-Consent-Flows.
- Admin autorisiert Scopes einmalig zentral in der Google Admin-Konsole.
- **Impersonation als Parameter:** Jede Operation erhält ein `impersonateAs`-Feld (Default = Funktionspostfach, Override pro Task).
- **Allowlist** der erlaubten Postfächer/Nutzer liegt in der Adapter-Konfiguration.
- **Service-Account-Key niemals im Repo** – ausschließlich über Secret-Store / Umgebungsvariablen.

### Referenz-Implementierungen
- **CIP-Adapter (live):** Node.js, `camunda-external-task-client-js`, Basic Auth via `got`-Interceptor-Pattern, Thema-Benennung mit Doppelpunkt-Trenner (`cip:resource:aktion`).
- **Asana-Adapter (geplant, eigenes Repo):** Gleiches Tech-Stack, Namensschema `asana:task:create` etc.
- **Google-Workspace-Adapter (dieses Repo):** Gleiches Tech-Stack, Namensschema `google:gmail:send-mail` etc.

### Infrastruktur
- **Operaton-Instanz:** orchestrator.mi-nautics.com (Hetzner Cloud, Docker Compose, Version 2.1.4)
- **Weitere Instanz:** uubato.com (separates Projekt)

---

## 2. Projektrichtlinien (vollständig)

Teil A und B gelten immer. Teil C nur bei öffentlicher API. Teil D nur bei eigener UI.

### Teil A: Allgemeine Regeln

#### A1 Release-Workflow
1. Arbeit auf einem Feature-Branch, Merge nach develop.
2. Stage-CI muss grün sein.
3. Claude liefert Testanweisungen für Björn.
4. Merge nach main und Deployment nach Prod erst nach ausdrücklicher Freigabe durch Björn.
5. Branch Protection auf main: kein Direkt-Push, Merge nur mit grüner CI und Freigabe.

Keine Ausnahmen, auch nicht für kleine Fixes.

#### A2 Hotfix-Pfad
Echte Produktionsstörungen und Sicherheitslücken nutzen einen verkürzten, aber nicht übersprungenen Pfad: Hotfix-Branch, CI grün, minimale Testanweisung, Freigabe durch Björn, danach Rückführung nach develop. Was als Hotfix gilt, entscheidet Björn.

#### A3 Wiederverwendung
Vor neuem Code prüfen, ob vorhandene Bausteine passen. Ein Muster wird in eine gemeinsame Funktion ausgelagert, sobald es 2–3 Mal vorkommt. Vorschnelle Abstraktion vermeiden.

#### A4 Business-Logik im Backend
Fachlogik liegt im Backend. Ist das im Einzelfall nicht machbar, vor der Umsetzung Rücksprache halten.

#### A5 Dokumentation und Protokollierung
- Entscheidungen und gewählte Ansätze stehen versioniert im Repo, z. B. in `DECISIONS.md` oder als ADRs.
- Zugänge: nur festhalten, wo und wie sie geregelt sind. Passwörter, Keys und Tokens landen weder im Memory noch im Repo.
- Kundensichtbare Features: Nach dem Prod-Release wird die Landingpage um Nutzenargumente ergänzt. Der Text geht vor Veröffentlichung zur Freigabe an Björn.

### Teil B: Sicherheit, Mandanten, Berechtigungen

#### B1 Nutzer, Mandant, Berechtigung
- Jede Backend-Route erhält den Nutzer (und daraus den Mandanten) und führt eine Berechtigungsprüfung aus.
- Der Mandantenfilter gehört in jede Datenbankabfrage, nicht nur in die Route.
- Der Test „fremder Mandant wird abgewiesen" ist Pflicht.

#### B2 Permission-Regel
- Fachliche Zugriffsprüfung im Backend läuft über `requirePerm('schluessel')`, im Frontend über `window.hasPerm('schluessel')`.
- `requireRole` ist nur für technische Admin-Operationen erlaubt.
- Im Frontend kein `me.role === …` zum Ein- und Ausblenden von UI.

#### B3 Reihenfolge bei neuen Features (pro PR, nicht pro Commit)
1. Migration
2. `PERM_KEYS`
3. `PERM_LABELS`
4. `PERM_HELP`
5. Route
6. Frontend
7. Tests

PR wird nicht gemergt, bevor alle 7 Schritte enthalten sind.

#### B4 Geheimnisse
Keys, Tokens und Passwörter gehören in die Secret-Verwaltung der Umgebung, nie in Code, Logs, Memory oder Testdaten.

### Teil C: Öffentliche API (nur bei externen Nutzern)

1. **Versionierung:** Nur additive Änderungen innerhalb einer Version. Alte Versionen bleiben mindestens 36 Monate erreichbar (zu bestätigen).
2. **Konsistenz:** Route, OpenAPI-Spezifikation und Beispiele im selben PR.
3. **Dokumentation ist Teil der Definition of Done.**
4. **Tests:** Erfolgsfall und alle Ablehnungsfälle.
5. **Muster:** Scope-Name `<objekt>:<aktion>`, Idempotenz über `external_id`, Fehlerformat `{ code, message, request_id }`.

### Teil D: Frontend und Design (nur bei eigener UI)

#### D1 Tokens statt Magic Numbers
- Farben, Schriftgrößen, Radien, Abstände aus CSS-Variablen.
- Raster: 4/8/12/16/24/32 px. Button-Padding: `8px 16px`.

#### D2 Struktur und Komponenten
- Genau ein `<h1>`, lückenlose Hierarchie, Landmarks (`<header>`, `<nav>`, `<main>`).

#### D3 Barrierefreiheit (WCAG 2.1 AA)
- Text-Kontrast ≥ 4,5:1; sichtbarer `:focus-visible`; Modals mit Fokus-Trap.

#### D4 Responsive
Mobile-first, drei Breakpoints (640/1024/1280 px).

#### D5 Mehrsprachigkeit
Keine hartkodierten UI-Strings; alles über Übersetzungsschlüssel.

### CI-Automatisierung
- Kein `requireRole` in geänderten Route-Dateien ohne Begründung
- Keine Direkt-Pushes auf main (Branch Protection)
- Keine Magic Numbers / fremde Farben (Stylelint)
- Keine hartkodierten UI-Strings (i18n-Lint)
- Barrierefreiheit (axe)
- OpenAPI aktuell (OpenAPI-Lint, Contract-Tests)
- Keine Geheimnisse im Repo (Secret-Scanning)

---

## 3. Offene Punkte (Stand 2026-10-03)

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
| 11 | Richtlinien | Gilt Teil C (öffentliche API) für dieses Projekt? |
| 12 | Richtlinien | Gilt Teil D (Frontend/Design) für dieses Projekt? |
| 13 | Später | Trigger-Modul Gmail (Cloud Pub/Sub oder Polling) |
| 14 | Später | Trigger-Modul Calendar (Watch-Channels oder Polling) |

---

## 4. Über Björn Richerzhagen (Kontext für Claude Code)

- **Name:** Björn Richerzhagen, Jahrgang 1978
- **Unternehmen:** MINAUTICS GmbH (Gründer & Geschäftsführender Consultant, seit Dezember 2010); mi-nautics.com
- **Standort:** Berlin
- **Hintergrund:** BPM-Consultant, Analyst und Trainer; Schwerpunkt BPMN, DMN, CMMN, Prozessautomatisierung (Camunda, Operaton, flowable, n8n, make)
- **Rolle in diesem Projekt:** Product Owner, Release-Freigabe
- **Wichtig:** Kein Direkt-Push auf main ohne seine ausdrückliche Freigabe!
- **Sprache:** Deutsch

---

## 5. Verwandte Projekte (Überblick)

| Projekt | Status | Tech |
|---|---|---|
| CIP-Adapter | live (privates Repo) | Node.js, camunda-external-task-client-js |
| Asana-Adapter | geplant (richerzhagen1978/operaton-adapter-asana) | Node.js |
| Google-Workspace-Adapter | in Planung (dieses Repo) | Node.js |

---

## 6. Operationen-Planung (vorläufig)

### Gmail-Modul
- `google:gmail:send-mail` – E-Mail senden (im Namen von)
- `google:gmail:reply-mail` – Auf E-Mail antworten
- `google:gmail:list-mails` – Postfach/Label durchsuchen
- `google:gmail:get-mail` – Einzelne E-Mail lesen
- `google:gmail:move-mail` – E-Mail in Label/Archiv verschieben
- `google:gmail:label-mail` – Label setzen/entfernen

### Calendar-Modul
- `google:calendar:create-event` – Termin anlegen
- `google:calendar:update-event` – Termin ändern
- `google:calendar:delete-event` – Termin löschen
- `google:calendar:get-event` – Termin lesen
- `google:calendar:list-events` – Termine eines Zeitraums listen

*(Umfang vor Implementierung mit Björn abstimmen)*

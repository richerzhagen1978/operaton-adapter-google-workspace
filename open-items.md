# Open Items & Projektkontext – operaton-adapter-google-workspace

Diese Datei dient Claude Code als vollständige Arbeitsgrundlage. Sie enthält alle gespeicherten Informationen über Projektkontext, Projektrichtlinien und offene Punkte.

---

## 1. Projektkontext

**Ziel:** Adapter für die BPMN-Prozessengine [Operaton](https://operaton.org), der Google-Workspace-Dienste anbindet – zunächst Gmail, später weitere Dienste (Kalender, Drive, Sheets u. a.).

**Repo:** https://github.com/richerzhagen1978/operaton-adapter-google-workspace

**Betreiber:** Björn Richerzhagen, MINAUTICS GmbH, Berlin

### Kernarchitektur (beschlossen)
- **Modularer Google-Workspace-Adapter** – kein Gmail-only-Adapter, kein separater Adapter je Produkt.
- **Core-Modul:** Auth (Domain-Wide Delegation via Service Account), Token-Refresh, Retry/Backoff bei Quotas, zentrale Konfiguration, Logging.
- **Service-Module:** Gmail, Calendar, Drive, Sheets usw. – je eigene Operationen und eigene Scopes, nur die konfigurierten Module werden geladen.
- **MVP-Scope:** Core + Gmail + Calendar; weitere Module erst bei konkretem Bedarf.

### Architekturgrundsatz (Kanal-Prinzip)
Operaton und Google Workspace kennen sich nicht – der Adapter kanalisiert alles:
- Operaton-Seite: Adapter spricht Operaton External Task API (Poll/FetchAndLock/Complete/Failure)
- Google-Seite: Adapter spricht Google-APIs (googleapis)

### Auth-Konzept (beschlossen)
- **Domain-Wide Delegation** mit einem Google Service Account.
- Ermöglicht Impersonation von Domain-Nutzern ohne interaktive OAuth-Consent-Flows.
- Admin autorisiert Scopes einmalig zentral in der Google Admin-Konsole.
- **Impersonation als Parameter:** Jede Operation erhält ein Feld "im Namen von" (E-Mail-Adresse). Default = Funktionspostfach, Override pro Task.
- **Allowlist:** Im Adapter konfigurierbar, welche Nutzer/Postfächer angesprochen werden dürfen.
- **Service-Account-Key:** Muss in Secret-Verwaltung der Umgebung liegen – NIEMALS in Code, Config-Dateien oder Logs.
- Bei höherer Last: Rate Limits / Quotas per Nutzer und Projekt beachten; Backoff im Core-Modul ist Pflicht.

### Anwendungsfall (noch offen)
Beide Richtungen sind denkbar:
- **Outbound:** Prozess sendet Mails / legt Termine an (einfacher, empfohlener MVP-Startpunkt)
- **Inbound (Trigger):** Prozess reagiert auf Ereignisse (neue Mail im Support-Postfach o. ä.) – deutlich aufwendiger, als eigenes Trigger-Modul später ergänzbar (Gmail: Cloud Pub/Sub; Calendar: Watch-Channels; alternativ Polling)

### Verknüpfung zum CIP-Adapter
Der Gmail-Adapter soll analog zum bestehenden CIP-Adapter gebaut werden (selbes Muster). CIP-Adapter-Code / -Details wurden im Chatverlauf noch nicht geteilt – müssen für Implementierungsstart bereitgestellt werden.

---

## 2. Projektrichtlinien (vollständig)

Diese Richtlinien gelten für alle Arbeiten an diesem Repo. **Teil A und B immer, Teil C und D nur wenn relevant.**

### Teil A: Allgemeine Regeln

#### A1 Release-Workflow
1. Arbeit auf einem Feature-Branch, Merge nach `develop`.
2. Stage-CI muss grün sein.
3. Claude liefert Testanweisungen für Björn.
4. Merge nach `main` und Deployment nach Prod **erst nach ausdrücklicher Freigabe durch Björn**.
5. Branch Protection auf `main`: kein Direkt-Push, Merge nur mit grüner CI und Freigabe.

**Keine Ausnahmen, auch nicht für kleine Fixes.**

#### A2 Hotfix-Pfad
Echte Produktionsstörungen und Sicherheitslücken nutzen einen verkürzten, aber nicht übersprungenen Pfad: Hotfix-Branch → CI grün → minimale Testanweisung → Freigabe durch Björn → Rückführung nach `develop`. Was als Hotfix gilt, entscheidet Björn.

#### A3 Wiederverwendung
- Vor neuem Code prüfen, ob vorhandene Bausteine passen.
- Ein Muster wird in eine gemeinsame Klasse/Funktion ausgelagert, sobald es zwei- bis dreimal vorkommt.
- Vorschnelle Abstraktion vermeiden.
- Ziel: keine Redundanz, konsistent und wartbar.

#### A4 Business-Logik im Backend
Fachlogik liegt im Backend. Ist das im Einzelfall nicht machbar, vor der Umsetzung Rücksprache halten.

#### A5 Dokumentation und Protokollierung
- Entscheidungen und gewählte Ansätze stehen versioniert im Repo, z. B. in `DECISIONS.md` oder als ADRs. Das Memory ergänzt, ersetzt es aber nicht.
- Zugänge: nur festhalten, wo und wie sie geregelt sind. Passwörter, Keys und Tokens landen weder in Memory noch im Repo.
- Kundensichtbare Features: Nach dem Prod-Release wird die Landingpage um Nutzenargumente ergänzt. Text geht vor Veröffentlichung zur Freigabe an Björn.

---

### Teil B: Sicherheit, Mandanten, Berechtigungen

#### B1 Nutzer, Mandant, Berechtigung
- Jede Backend-Route erhält den Nutzer (und daraus den Mandanten) und führt eine Berechtigungsprüfung aus.
- Der Mandantenfilter gehört in jede Datenbankabfrage, nicht nur in die Route.
- Der Test „fremder Mandant wird abgewiesen" ist Pflicht.

#### B2 Permission-Regel
- Fachliche Zugriffsprüfung im Backend: `requirePerm('schluessel')` / im Frontend: `window.hasPerm('schluessel')`.
- `requireRole` nur für technische Admin-Operationen: Nutzerverwaltung, Berechtigungsmatrix, System-Config.
- Im Frontend kein `me.role === …` zum Ein-/Ausblenden von UI. Ausnahme: negative Abgrenzung von Gast- und Kundenrollen (`me.role === 'gast'`).

#### B3 Reihenfolge bei neuen Features (pro Feature/PR, nicht pro Commit)
1. Migration
2. `PERM_KEYS` (z. B. `tenant.js`)
3. `PERM_LABELS`
4. `PERM_HELP`
5. Route
6. Frontend
7. Tests

Ein Pull Request wird erst gemergt, wenn alle sieben Schritte enthalten sind. Vor dem Merge prüfen: Kommt `requireRole` in geänderten Route-Dateien vor → begründen oder durch `requirePerm` ersetzen.

#### B4 Geheimnisse
Keys, Tokens und Passwörter gehören in die Secret-Verwaltung der Umgebung – **niemals in Code, Logs, Memory oder Testdaten**.

---

### Teil C: Öffentliche API (nur bei externen Nutzern)

Die öffentliche API ist ein Produkt: Innerhalb einer Version nur additive Änderungen.

1. **Versionierung:** Neue optionale Felder und Endpunkte sind additiv. Feld entfernen, Typ ändern, Pflichtfeld ergänzen oder Statuscode ändern → neue Version. Alte Versionen bleiben mindestens [Laufzeit festlegen, Original: 36 Monate] erreichbar.
2. **Konsistenz:** Feld ergänzt/umbenannt → Route, OpenAPI-Spezifikation und Beispiele im selben PR.
3. **Dokumentation ist Teil der Definition of Done:** Kein API-Merge ohne aktualisierte OpenAPI-Spezifikation; bei neuen Endpunkten zusätzlich ein lauffähiges Beispiel.
4. **Tests:** Erfolgsfall UND Ablehnungsfälle (falscher Schlüssel, fehlender Scope, fremder Mandant, doppelte `external_id`, Ratenlimit). Nur Erfolgsfall getestet = ungetestet.
5. **Muster für neue Objekttypen:** Scope-Name `<objekt>:<aktion>`, Idempotenz über `external_id`, Fehlerformat mit `code`, `message`, `request_id`. Abweichungen nur mit Begründung im Konzeptdokument.

---

### Teil D: Frontend und Design (nur bei eigener UI)

CIP-Markenfarben und -Tokens (#123274, #E94D65 usw.) sind CIP-spezifisch – nicht automatisch auf dieses Projekt übertragen.

#### D1 Tokens statt Magic Numbers
- Farben, Schriftgrößen, Radien, Abstände, Schatten aus CSS-Variablen. Neue Werte nur nach Erweiterung des Token-Systems, nie inline.
- Abstände nur aus dem Raster 4 / 8 / 12 / 16 / 24 / 32 px. Button-Padding: `8px 16px`.
- Statusfarben vor Übernahme mit Kontrastchecker prüfen (Text mindestens 4,5:1 auf Weiß).

#### D2 Struktur und Komponenten
- Genau ein `<h1>` je Seite, lückenlose Hierarchie h1 → h2 → h3, semantische Tags.
- Landmarks: ein `<header>`, `<nav>`, `<main>`, ggf. `<footer>`; Skip-Link zum Hauptinhalt.
- Gleiche Dinge sehen gleich aus: eine Primäraktion pro Sicht, Sekundär- und Ghost-Buttons einheitlich.

#### D3 Barrierefreiheit (WCAG 2.1 AA)
- Text-Kontrast ≥ 4,5:1, große Schrift und UI-Elemente ≥ 3:1; Farbe nie als einziger Bedeutungsträger.
- Sichtbarer `:focus-visible`-Fokus; Tastaturbedienung; Modals mit Fokus-Trap und Esc.
- `alt` an jedem Bild, Label/`aria-label` an jedem Formularfeld, `prefers-reduced-motion` respektieren.
- Touch-Ziele mindestens 44 × 44 px (interne Regel).

#### D4 Responsive
Mobile-first, drei Breakpoints (640 / 1024 / 1280 px), Datentabellen in scrollbarem Wrapper, kein horizontaler Body-Overflow.

#### D5 Mehrsprachigkeit
Keine hartkodierten UI-Strings; alles über Übersetzungsschlüssel (inkl. Platzhalter, Tooltips, Leerzustände, aria-labels). Sprachwechsel aktualisiert `html[lang]`. Pluralisierung über i18n-Regeln, Datums-/Zahlenformate über Intl-API.

---

### CI-Automatisierung
Mechanische Regeln als CI-Checks:
- Kein `requireRole` in geänderten Route-Dateien ohne Begründung (Skript oder Lint-Regel)
- Keine Direkt-Pushes auf `main` (Branch Protection)
- Keine Magic Numbers und fremden Farben (Stylelint)
- Keine hartkodierten UI-Strings (i18n-Lint)
- Barrierefreiheit (axe)
- OpenAPI aktuell, Verträge eingehalten (OpenAPI-Lint, Contract-Tests)
- Keine Geheimnisse im Repo (Secret-Scanning)

---

## 3. Offene Punkte / To-do

| # | Bereich | Beschreibung | Status |
|---|---------|-------------|--------|
| 1 | Architektur | CIP-Adapter-Code/-Details bereitstellen, damit Gmail-Adapter analog gebaut werden kann | ⏳ Offen |
| 2 | Auth | Service Account anlegen und Domain-Wide Delegation in Google Admin-Konsole konfigurieren | ⏳ Offen |
| 3 | Auth | Secret-Store für Service-Account-Key festlegen und konfigurieren | ⏳ Offen |
| 4 | Core | Allowlist-Konzept für erlaubte Nutzer/Postfächer definieren | ⏳ Offen |
| 5 | Anwendungsfall | Entscheiden: Outbound-only MVP oder auch Inbound-Trigger | ⏳ Offen |
| 6 | Repo | Branch `develop` anlegen, Branch Protection auf `main` konfigurieren | ⏳ Offen |
| 7 | Repo | CI-Pipeline aufsetzen (mind. Lint, Tests, Secret-Scanning) | ⏳ Offen |
| 8 | Implementierung | Core-Modul: Auth, Token-Refresh, Retry/Backoff, Konfiguration, Logging | ⏳ Offen |
| 9 | Implementierung | Service-Modul Gmail: Mail senden (Outbound MVP) | ⏳ Offen |
| 10 | Implementierung | Service-Modul Calendar: Termine anlegen (MVP) | ⏳ Offen |
| 11 | Inbound (optional) | Trigger-Modul: Gmail via Cloud Pub/Sub oder Polling | ⏳ Später |
| 12 | Inbound (optional) | Trigger-Modul: Calendar via Watch-Channels oder Polling | ⏳ Später |

---

## 4. Technische Hinweise für Claude Code

- **Technologie-Stack:** Noch nicht festgelegt – CIP-Adapter-Vorlage abwarten. Vermutlich Node.js (External Task Worker Muster).
- **Google-Bibliothek:** Offizielle `googleapis` npm-Packages (aktuelle Version und exakte Package-Namen in der Google-Doku / npm verifizieren).
- **Operaton External Task API:** Endpunkte `/external-task/fetchAndLock`, `/complete`, `/failure` – Details in der Operaton-Doku verifizieren.
- **Scopes:** So eng wie möglich halten; aktuelle Scope-Bezeichnungen in der Google-API-Doku verifizieren.
- **Kein Geheimnis im Repo:** Service-Account-Key, OAuth-Tokens, API-Keys ausschließlich über Umgebungsvariablen / Secret-Store einbinden.
- **DECISIONS.md:** Architekturentscheidungen dort dokumentieren, nicht nur im Memory.

---

*Zuletzt aktualisiert: 2026-10-03*

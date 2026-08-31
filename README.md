# 📚 StudyBuddy

**Ein leichtgewichtiger Lernplaner: Fächer, Lernziele, Lerneinheiten und Fortschritt an einem Ort.**

StudyBuddy ist eine kleine Web-App für Schüler, Studierende und autodidaktische Lernende. Sie läuft als **eine einzelne statische Binary** mit einer SQLite-Datei daneben. Keine Runtime, kein Container-Zwang, kein Setup-Marathon.

[![CI](https://github.com/<DEIN-GITHUB-NAME>/StudyBuddy/actions/workflows/ci.yml/badge.svg)](https://github.com/<DEIN-GITHUB-NAME>/StudyBuddy/actions)
![Go](https://img.shields.io/badge/Go-1.22%2B-00ADD8)
![License](https://img.shields.io/badge/license-MIT-blue)

<!-- TODO: Screenshot einfügen, sobald M4 steht -->
<!-- ![StudyBuddy Dashboard](docs/screenshot-dashboard.png) -->

---

## Über das Projekt

Lernplanung scheitert selten am Wollen, sondern an fehlender Übersicht: Wie viel habe ich diese Woche tatsächlich gemacht? Welches Ziel läuft mir gerade davon?

StudyBuddy beantwortet genau diese zwei Fragen und sonst nichts. Alle Fortschrittswerte werden aus den erfassten Lerneinheiten **berechnet**, nicht redundant gespeichert.

---

## Warum Go statt ASP.NET Core?

Die erste Version dieses Projekts war eine ASP.NET-Core-Anwendung mit Razor Pages, EF Core und Bootstrap 5. Der alte Stand ist als Tag [`v0.1-dotnet`](../../releases/tag/v0.1-dotnet) erhalten.

Der Wechsel hatte drei Gründe:

1. **Deployment-Aufwand.** Ziel war eine App, die auf einem 3-Euro-VPS ohne Laufzeitumgebung läuft. Go kompiliert nach `CGO_ENABLED=0` zu einer statischen Binary; das Container-Image liegt unter 20 MB.
2. **Verhältnis von Framework zu Anwendung.** Der fachliche Kern der App sind ein paar CRUD-Masken und ein Planungsalgorithmus. Dafür reichen `net/http`, `html/template` und `database/sql` aus der Standardbibliothek. Kein Router-Framework, kein ORM, kein CSS-Framework.
3. **Testbarkeit der Kernlogik.** Der Wochenplan-Generator ist als reine Funktion ohne DB- oder HTTP-Abhängigkeiten implementiert und dadurch mit Tabellentests vollständig abgedeckt.

Die Begründungen zu einzelnen Technikentscheidungen liegen als Architecture Decision Records unter [`docs/adr/`](docs/adr/).

---

## Features

**Legende:** ✅ fertig · 🟡 in Arbeit · ⬜ geplant

### v1.0 – MVP

| Status | Feature |
|:---:|---|
| ⬜ | Fächer anlegen, bearbeiten, löschen |
| ⬜ | Lernziele je Fach mit Zielumfang und Deadline |
| ⬜ | Lerneinheiten erfassen (Datum, Dauer, Notiz) |
| ⬜ | Fortschrittsanzeige je Ziel und Wochenübersicht |
| ⬜ | Serverseitige Validierung, funktioniert ohne JavaScript |

### v1.1 und später

| Status | Feature |
|:---:|---|
| ⬜ | Automatische Wochenplan-Erstellung (Greedy-Verteilung nach Dringlichkeit) |
| ⬜ | Hinweis auf fällige Lerneinheiten in den nächsten 48 h |
| ⬜ | Optionale E-Mail-Erinnerung per SMTP |
| ⬜ | Authentifizierung für mehrere Benutzer |
| ⬜ | Deployment auf Fly.io oder VPS |

---

## Tech Stack

| Bereich | Technologie | Warum |
|---|---|---|
| Sprache | Go 1.22+ | Statische Binary, schnelle Builds |
| HTTP | `net/http` (`ServeMux`) | Ab Go 1.22 mit Methoden- und Pfad-Patterns, kein Router nötig |
| Templates | `html/template` + `embed` | Auto-Escaping, alles in der Binary |
| Datenbank | SQLite via `modernc.org/sqlite` | Pure Go, kein CGo |
| Migrationen | Eingebettete `.sql`-Dateien, beim Start ausgeführt | Kein zusätzliches Tool |
| Styling | Handgeschriebenes CSS | Kein Framework-Ballast für vier Seiten |
| Diagramme | Inline-SVG / CSS-Balken | Fortschrittsbalken brauchen keine Chart-Library |
| Tests | `testing`, `net/http/httptest` | Standardbibliothek |
| CI | GitHub Actions (`go vet`, `go test`, `golangci-lint`) | |

---

## Schnellstart

**Voraussetzung:** Go 1.22 oder neuer.

```bash
git clone https://github.com/<DEIN-GITHUB-NAME>/StudyBuddy.git
cd StudyBuddy
go run ./cmd/studybuddy
```

Die App läuft anschließend auf <http://localhost:8080>. Beim ersten Start legt sie `studybuddy.db` im Arbeitsverzeichnis an und führt die Migrationen aus.

### Konfiguration

Alles über Umgebungsvariablen, alles mit sinnvollem Default:

| Variable | Default | Bedeutung |
|---|---|---|
| `SB_ADDR` | `:8080` | Adresse und Port |
| `SB_DB` | `studybuddy.db` | Pfad zur SQLite-Datei |

### Build

```bash
CGO_ENABLED=0 go build -ldflags="-s -w" -o studybuddy ./cmd/studybuddy
```

---

## Projektstruktur

```
cmd/studybuddy/      Einstiegspunkt, Konfiguration, Serverstart
internal/store/      Datenbankzugriff, SQL-Queries
internal/web/        HTTP-Handler, Templates, statische Dateien
internal/planner/    Wochenplan-Logik (reine Funktionen, ohne DB und HTTP)
migrations/          Eingebettete SQL-Migrationen
docs/adr/            Architecture Decision Records
```

---

## Datenmodell

```sql
CREATE TABLE subjects (
  id         INTEGER PRIMARY KEY,
  name       TEXT NOT NULL,
  color      TEXT NOT NULL DEFAULT '#4f46e5',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE goals (
  id             INTEGER PRIMARY KEY,
  subject_id     INTEGER NOT NULL REFERENCES subjects(id) ON DELETE CASCADE,
  title          TEXT NOT NULL,
  target_minutes INTEGER NOT NULL DEFAULT 0,
  due_date       DATE,
  done           BOOLEAN NOT NULL DEFAULT 0
);

CREATE TABLE sessions (
  id         INTEGER PRIMARY KEY,
  subject_id INTEGER NOT NULL REFERENCES subjects(id) ON DELETE CASCADE,
  goal_id    INTEGER REFERENCES goals(id) ON DELETE SET NULL,
  started_at DATETIME NOT NULL,
  minutes    INTEGER NOT NULL CHECK (minutes > 0),
  note       TEXT
);
```

Der Fortschritt eines Ziels ist keine Spalte, sondern das Ergebnis von `SUM(sessions.minutes) / goals.target_minutes`.

---

## Wochenplan-Algorithmus

```go
func PlanWeek(goals []Goal, slots []TimeSlot, now time.Time) []PlannedSession
```

Die Ziele werden nach Dringlichkeit sortiert (verbleibende Minuten geteilt durch verbleibende Tage bis zur Deadline) und der Reihe nach in die verfügbaren Zeitfenster gefüllt. Die Funktion hat keine Seiteneffekte und ist über Tabellentests abgedeckt.

---

## Tests

```bash
go test ./...
go vet ./...
```

---

## Deployment

Multi-Stage-Build mit `scratch` beziehungsweise `distroless/static` als Basis:

```bash
docker build -t studybuddy .
docker run -p 8080:8080 -v studybuddy-data:/data -e SB_DB=/data/studybuddy.db studybuddy
```

> **Wichtig:** Die SQLite-Datei muss auf einem persistenten Volume liegen, sonst sind die Daten beim nächsten Deploy verloren. Für Backups genügt ein regelmäßiges `sqlite3 .backup` per Cron.

---

## Roadmap

| Meilenstein | Inhalt | Fertig, wenn |
|---|---|---|
| M0 | Repo-Setup, Modul, Ordnerstruktur | Server startet auf `:8080` |
| M1 | DB-Layer, Migrationen, Repository für Fächer | Test legt ein Fach an und liest es zurück |
| M2 | Routing, Layout, Fächerliste | `/subjects` zeigt echte Daten |
| M3 | CRUD für Fächer und Ziele inkl. Validierung | Anlegen, Ändern, Löschen ohne JavaScript |
| M4 | Lerneinheiten und Fortschrittsbalken | Startseite zeigt Wochensumme und Balken je Ziel |
| M5 | Tests, Linting, CI | Grüner Badge im README |
| M6 | Dockerfile und Deployment | Öffentlich erreichbare URL |
| M7 | Wochenplan-Generator | Reine Funktion mit Tests, dann UI |
| M8 | Authentifizierung | Login, Session-Cookie, alle Queries benutzerbezogen |

---

## Lizenz

MIT. Siehe [LICENSE](LICENSE).

---

*Stand: 31.08.2026*

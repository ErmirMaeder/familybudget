# FamilyBudget — Projektdokumentation

Familienbudget-App als Progressive Web App (PWA) mit Cloud-Backend über Supabase.
Apple-Design-Sprache (iOS-Grouped-Lists, Bottom-Sheets, System-Tab-Bar).

## Inhalt dieses Ordners

| Datei | Zweck |
|---|---|
| `index.html` | Die komplette App — HTML, CSS und JavaScript in einer Datei |
| `manifest.json` | PWA-Manifest (Name, Icons, Farben, "Zum Home-Bildschirm") |
| `sw.js` | Service Worker fürs Offline-Caching |
| `icon-180.png` | App-Icon für iPhone Home-Bildschirm (180×180) |
| `icon-192.png` | App-Icon für Android/PWA (192×192) |
| `icon-512.png` | App-Icon groß (512×512, App-Store-Auflösung) |

**Alle Dateien müssen im selben Ordner/Repo-Root liegen** (z.B. GitHub Pages Root), da `index.html` sie über relative Pfade referenziert (`./manifest.json`, `./sw.js`, `./icon-*.png`).

---

## Tech-Stack

- **Frontend:** Vanilla JavaScript (kein Framework), einzelne HTML-Datei
- **Styling:** Reines CSS mit CSS-Variablen, iOS-Design-System nachgebaut (SF-Pro-ähnliche Typografie, `-apple-system` Font-Stack)
- **Backend:** [Supabase](https://supabase.com) (PostgreSQL + Auth + Realtime, kostenloser Tier)
  - Auth: E-Mail/Passwort via `supabase-js` v2 (CDN: `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2`)
  - Datenbank: 4 Tabellen (`families`, `profiles`, `budget_data`, `recurring`)
- **Charts:** Chart.js v4.4.1 (CDN: `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js`)
- **Hosting:** GitHub Pages (statisch, kein Server nötig)
- **Dark Mode:** automatisch via `@media (prefers-color-scheme: dark)`

---

## Supabase-Konfiguration

Die Zugangsdaten stehen direkt im Code (Zeile ~1140 in `index.html`):

```js
const SUPA_URL = 'https://azzrtusnfxiraerdmfkm.supabase.co';
const SUPA_KEY = 'eyJhbGciOiJIUzI1NiIs...'; // anon/public key, NICHT der secret key
```

⚠️ **Wichtig:** Nur der `anon`/`public` Key darf im Frontend stehen, niemals der `service_role`/`secret` Key (Sicherheitsrisiko).

### Datenbank-Schema

```sql
-- Familien (jede Familie = eine geteilte Datenwelt)
create table families (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  invite_code text unique not null default upper(substring(gen_random_uuid()::text, 1, 6)),
  created_at timestamptz default now()
);

-- Benutzer-Profile (verknüpft Supabase-Auth-User mit einer Familie)
create table profiles (
  id uuid primary key references auth.users on delete cascade,
  username text unique not null,
  family_id uuid references families(id),
  role text default 'member', -- 'admin' oder 'member'
  created_at timestamptz default now()
);

-- Budget-Daten (pro Familie + Monat, JSON-Blob)
create table budget_data (
  id uuid primary key default gen_random_uuid(),
  family_id uuid references families(id) on delete cascade not null,
  month_key text not null, -- Format: 'YYYY-MM', z.B. '2026-07'
  data jsonb not null default '{}', -- {income:[...], expenses:[...], closed:bool}
  updated_at timestamptz default now(),
  unique(family_id, month_key)
);
-- Sonderfall: month_key = '__settings__' speichert Kategorien (data.cats)

-- Daueraufträge & wiederkehrende Einnahmen + Personen (ein Datensatz pro Familie)
create table recurring (
  id uuid primary key default gen_random_uuid(),
  family_id uuid references families(id) on delete cascade not null,
  data jsonb not null default '[]', -- {rec:[...], recInc:[...], persons:[...]}
  updated_at timestamptz default now(),
  unique(family_id)
);
```

### Row Level Security (RLS) — wichtig für Bugfixing

Alle 4 Tabellen haben RLS aktiviert. Policies sind bewusst **nicht rekursiv** aufgebaut
(frühere Version hatte "infinite recursion" Fehler wegen verschachtelter Subqueries auf `profiles`).

Aktuelle funktionierende Policies (vereinfacht):
- `profiles`: SELECT `using(true)` (jeder eingeloggte Nutzer darf alle Profile lesen — unkritisch, da keine sensiblen Daten)
- `families`: SELECT/INSERT offen für eingeloggte Nutzer
- `budget_data`/`recurring`: nur Zugriff wenn `family_id` zum eigenen Profil passt

Falls neue RLS-Fehler auftreten → Supabase SQL Editor → `select policyname, tablename from pg_policies;` prüfen.

---

## App-Architektur (index.html)

### Bildschirm-Reihenfolge beim Start

1. **Splash-Screen** (`#splash`) — kurzer Ladebildschirm mit Logo
2. **Login-Screen** (`#loginScreen`) — Anmelden / Registrieren / Passwort vergessen
3. **Familie-Setup** (`#familyScreen`) — nur beim ersten Login: Familie erstellen oder per Code beitreten
4. **Monat-Auswahl** (`#monthSelectScreen`) — nach jedem Login: Monat wählen bevor man in die App kommt
5. **Hauptapp** (`#app`) — Dashboard, Einnahmen, Ausgaben, Jahresübersicht, Statistiken, Einstellungen, Hilfe

### Navigation

- **Desktop (≥768px):** horizontale Tab-Leiste oben (`.desk-nav`), scrollbar
- **Mobile (<768px):** iOS-artige Bottom-Tab-Bar (`.ios-tab-bar`) mit 5 Tabs: Übersicht, Einnahmen, Ausgaben, Jahres, Mehr (Mehr = Einstellungen)
- **Home-Button** (🏠 oben links) auf jeder Seite → zurück zur Monat-Auswahl

### Seiten (`.page` divs, gesteuert über `go(pageName)`)

| ID | Name | Zeigt |
|---|---|---|
| `pg-dash` | Dashboard | Kennzahlen, Charts (Kategorie-Pie, Einnahmen/Ausgaben-Bar), Sparquote, Top-Kategorien, letzte Ausgaben, Budget-Aufteilung pro Person |
| `pg-income` | Einnahmen | Liste + wiederkehrende Einnahmen (Lohn etc.) |
| `pg-expenses` | Ausgaben | Liste + wiederkehrende Ausgaben/Daueraufträge (Miete, Abos etc.) — **beide zusammengelegt in einer Seite** |
| `pg-overview` | Jahresübersicht | Balkendiagramm + Monatsliste für ein ganzes Jahr |
| `pg-stats` | Statistiken | Sparverlauf-Chart, Top-Kategorien, CSV-Export |
| `pg-settings` | Einstellungen | Personen, Kategorien (mit Emoji-Picker), Zahlungsarten, Konto/Familie, Auto-Logout-Timer |
| `pg-help` | Hilfe | Statische Anleitung |

### Kern-Datenstrukturen (globale JS-Variablen)

```js
let D = {};              // { "2026-07": {income:[...], expenses:[...], closed:false}, ... }
let REC = [];             // Daueraufträge (wiederkehrende Ausgaben) — Templates
let REC_INC = [];         // wiederkehrende Einnahmen — Templates
let PERSONS = [];         // ["Ermir", "Larissa"] — für Budget-Aufteilung
let CATS = {exp:[...], inc:[...], pay:[...]}; // Kategorien, exp/inc als {n:name, e:emoji}
let currentUser, currentProfile, currentFamily; // Supabase-Session-Objekte
let cy, cm;               // aktuell angezeigtes Jahr/Monat (current year/month)
```

### Wichtige Funktionsgruppen

**Auth** (`doLogin`, `doRegister`, `doReset`, `doLogout`, `onLogin`)
**Familie** (`createFamily`, `joinFamily`, `switchFamily`)
**Daten laden/speichern** (`loadAllData`, `saveMonthData`, `saveCats`, `saveRecurring`, `save()` — debounced mit 800ms)
**Rendering pro Seite** (`rDash`, `rInc`, `rExp`, `rOv`, `rStats`, `rSettings`, `rRec`)
**Daueraufträge/wiederkehrende Einnahmen** — **kein Auto-Booking mehr!** Nutzer muss manuell "Übernehmen" klicken (`transferAllRecExp`, `transferAllRecInc`, `bookNow`, `bookRecInc`)
**Monat abschliessen** (`isMonthClosed`, `toggleMonthClosed`) — sperrt Bearbeiten/Löschen für einen Monat bis reaktiviert
**Eigener Bestätigungsdialog** (`showConfirm`) — **wichtig:** `window.confirm()` wird von iOS in installierten PWAs (Standalone-Modus) stillschweigend unterdrückt und liefert sofort `false` zurück. Deshalb NIE `confirm()` verwenden, immer `await showConfirm(message, buttonLabel)`.
**Auto-Logout** (`startInactivityTimer`, `resetInactivityTimer` via `markActivity`) — zeitstempelbasiert mit `setInterval`, NICHT nur `setTimeout` (das würde in Hintergrund-Tabs gedrosselt/pausiert)
**Kategorien mit Emojis** (`catN()` normalisiert Kategorie zu String-Namen — wichtig, da Kategorien intern als `{n, e}`-Objekte gespeichert werden, aber überall als reiner String angezeigt werden müssen)

---

## Bekannte Design-Entscheidungen / Fallstricke für künftige Änderungen

1. **`catN(cat)` IMMER verwenden**, wenn eine Kategorie angezeigt oder verglichen wird — sie kann String oder `{n,e}`-Objekt sein, je nachdem woher sie kommt.
2. **`confirm()` und `alert()` vermeiden** — iOS PWA Standalone-Modus blockiert sie. Stattdessen `showConfirm()` (bereits vorhanden) bzw. den `toast()`/Sheet-Mechanismus nutzen.
3. **Kein Auto-Booking mehr** — Daueraufträge/wiederkehrende Einnahmen werden nie automatisch in einen Monat eingetragen. Nutzer muss "Übernehmen" klicken. Das war eine bewusste Änderung (vorher gab es Probleme mit veralteten automatisch gebuchten Werten).
4. **Monat-Schliessen-Funktion** — `D[key].closed` sperrt Bearbeiten/Löschen. Beim Hinzufügen neuer Features die Einnahmen/Ausgaben verändern, immer `isMonthClosed()` am Anfang prüfen.
5. **Syntax-Fehler-Falle:** Bei mehreren Bearbeitungsrunden ist mehrfach eine fehlende `function name(){` Deklaration aufgetreten (Copy-Paste-Fehler beim Ersetzen). Vor jedem Deployment sollte der Code mit folgendem Snippet auf Syntaxfehler geprüft werden:
   ```bash
   python3 -c "
   import re
   content = open('index.html').read()
   scripts = re.findall(r'<script>(.*?)</script>', content, re.DOTALL)
   open('/tmp/t.js','w').write('\n'.join(scripts))
   " && node /tmp/t.js
   # Einzige erwartete Fehlermeldung: "ReferenceError: supabase is not defined"
   # (weil supabase-js nur im Browser existiert). Jeder ANDERE Fehler ist ein echter Bug.
   ```
6. **`showSyncPill(state)`** aktualisiert zwei mögliche Sync-Dots (`#syncDot` im Nav, `#syncDotSettings` in den Einstellungen) — beide müssen mit `if(dot)` abgesichert bleiben, da nicht auf jeder Seite beide Elemente existieren.

---

## Deployment (GitHub Pages)

1. Alle 6 Dateien (`index.html`, `manifest.json`, `sw.js`, `icon-180.png`, `icon-192.png`, `icon-512.png`) in ein GitHub-Repo hochladen — direkt im Root, nicht in einem Unterordner
2. Repo-Einstellungen → Pages → Source: "Deploy from a branch" → `main` → `/ (root)`
3. Nach 1–2 Minuten live unter `https://<username>.github.io/<repo-name>/`
4. Auf iPhone: Safari öffnen → Teilen-Symbol → "Zum Home-Bildschirm"

## Was noch fehlt / mögliche nächste Schritte

- Kein natives App-Store-Listing (aktuell reine PWA) — für echten App Store bräuchte es Capacitor + Apple Developer Account (99$/Jahr)
- Keine Push-Benachrichtigungen (iOS erlaubt das nur eingeschränkt für PWAs seit iOS 16.4)
- E-Mail-Bestätigung bei Registrierung ist in Supabase aktuell deaktiviert (Auth → Providers → Email → "Confirm email" aus) — für Produktivbetrieb evtl. wieder aktivieren
- Registrierung ist aktuell offen für alle — schützt sich nur durch den Familien-Einladungscode, nicht durch Zugriffsbeschränkung auf Supabase-Ebene

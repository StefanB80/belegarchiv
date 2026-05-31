# Belegarchiv (Meta)

Dieses Repository ist der **Meta-Einstieg** (Dokumentation, Regeln, lokaler Gesamt-Checkout) — **kein** Deploy-Artefakt des Anwendungskerns. Architektur analog [APInterface Modulentwicklung](https://apinterface.com/docs/modulentwicklung): **Core**, **Platform** und **Module** sind **eigene Git-Repositories** mit eigener Versionierung.

## Repository-Zuordnung (Variante 1)

| Repository | GitHub (Beispiel) | Inhalt |
|------------|-------------------|--------|
| **Meta** | `StefanB80/belegarchiv` | Dieses Repo: README, `.cursor/rules`, Submodule-Refs, ggf. Gesamt-Skripte |
| **Core** | `StefanB80/belegarchiv-core` *(anlegen)* | API, Prisma, Auth, Modul-Loader — **Repo-Root = heutiger Ordner `core/`** |
| **Platform** | `StefanB80/belegarchiv-platform` *(anlegen)* | Compose, Nginx, Deploy-Hooks, `DEPLOY.md` |
| **Modul UID** | [`StefanB80/belegarchiv-module-UID`](https://github.com/StefanB80/belegarchiv-module-UID) | CHE/UID im Mandanten-Kontext; Deploy per Actions **deploy-module-uid-vserver**, Secrets **`BAU_*`** |
| **Modul (Beispiel)** | [`StefanB80/belegarchiv-module-example-app`](https://github.com/StefanB80/belegarchiv-module-example-app) *(optional)* | Semver-Ordner (`0.1.0/`, …), `moduleManifest.json`, `index.js`; Actions **deploy-module-example-vserver**, **`BAM_*`** |
| **Modul Belege** | [`StefanB80/belegarchiv-module-belege`](https://github.com/StefanB80/belegarchiv-module-belege) *(anlegen)* | Belegverwaltung als eigenes Modul (`appKey` **belege**); Actions **deploy-module-belege-vserver**, Secrets **`BAB_*`** |
| **Modul OCR** | [`StefanB80/belegarchiv-module-ocr`](https://github.com/StefanB80/belegarchiv-module-ocr) *(anlegen)* | Volltextsuche in Belegen (`appKey` **ocr**); Actions **deploy-module-ocr-vserver**, Secrets **`BAO_*`** |

## Erst-Einrichtung (wenn die Ziel-Repos leer sind)

1. **Core-Repo** `belegarchiv-core` auf GitHub anlegen.  
   Lokal im Ordner `core/`: `git init`, Remote setzen, `main` pushen (Inhalt = jetziger `core/`-Tree inkl. `.github/workflows/`).
2. **Platform-Repo** `belegarchiv-platform` anlegen, Inhalt = jetziger Ordner `platform/` pushen.
3. **Modul-Repo(s):** z. B. **UID** ([belegarchiv-module-UID](https://github.com/StefanB80/belegarchiv-module-UID), produktiver Actions-Deploy) und optional **Example** — Inhalt = jeweiliger Modulbaum mit SemVer-Unterordner.
4. **Meta-Repo** (dieses): `.gitmodules.example` nach `.gitmodules` kopieren, URLs prüfen, dann:

   ```bash
   git submodule update --init --recursive
   ```

   Alternativ Submodule nachträglich hinzufügen:

   ```bash
   git submodule add https://github.com/StefanB80/belegarchiv-core.git core
   git submodule add https://github.com/StefanB80/belegarchiv-platform.git platform
   ```

   Für das UID-Modul (optional, lokaler Meta-Checkout):

   ```bash
   git submodule add https://github.com/StefanB80/belegarchiv-module-UID.git modules/uid
   ```

   Für ein Beispielmodul (optional):

   ```bash
   git submodule add https://github.com/StefanB80/belegarchiv-module-example-app.git modules/example-app
   ```

   Für das Modul **Belege** (optional):

   ```bash
   git submodule add https://github.com/StefanB80/belegarchiv-module-belege.git modules/belege
   ```

   Für das Modul **OCR** (optional):

   ```bash
   git submodule add https://github.com/StefanB80/belegarchiv-module-ocr.git modules/ocr
   ```

Bis die Submodule stehen, kannst du lokal weiterhin im **gemeinsamen Workspace** (alle Ordner nebeneinander) arbeiten — die Aufteilung ist die verbindliche Zielstruktur für Git und vServer.

**Wichtig für den ersten Push ins Meta-Repo:** Nur Meta-Dateien committen (`README.md`, `.cursor/`, `.gitmodules.example` …). Ordner `core/`, `platform/`, `modules/` **nicht** ins Meta-Repo aufnehmen, bevor die jeweiligen Ziel-Repos existieren und per `git submodule add …` angebunden sind — sonst entsteht ein doppelter Codepfad ohne Submodule.

## Deploy

Operative Anleitung: Repository **Platform** → Datei [`DEPLOY.md`](https://github.com/StefanB80/belegarchiv-platform/blob/main/DEPLOY.md) (GitHub Actions Secrets, systemd, Bootstrap-Skript).

**GitHub Secrets** kann ich von hier aus nicht setzen (`gh` fehlt, kein Token) — im Repo **belegarchiv-core** unter *Settings → Secrets and variables → Actions* anlegen (Namen siehe `DEPLOY.md`).

## Hybrid-Module (Remote-Dienste)

Zusätzlich zu eingebetteten Modulen unter `modules/<appKey>/<semver>/` kann der Katalog **Remote-Apps** führen, die **nicht** auf dem Core-Server liegen (eigener Dienst, eigener Release-Zyklus).

- **Konfiguration (ohne DB):** Umgebungsvariable `BELEGARCHIV_REMOTE_MODULES` (JSON-Array) und/oder `BELEGARCHIV_REMOTE_MODULES_FILE` (Pfad zu JSON, siehe `core/config/remote-modules.example.json`). Nur **https**-`baseUrl`, außer `BELEGARCHIV_ALLOW_INSECURE_REMOTE=1` (Entwicklung).
- **Katalog:** `GET /api/v1/catalog/modules` liefert `integrationKind` (`embedded` \| `remote`) und bei Remote `remoteBaseUrl`.
- **Konflikt:** Gleicher `appKey` auf Disk und in Remote-Config: **Disk-Modul gewinnt**.
- **Ping:** `GET /api/v1/companies/:companyId/apps/:appKey/ping` — eingebettete Module per Loopback (`register`), Remote per `fetch` auf `{remoteBaseUrl}{remotePingPath}` (Default `/belegarchiv/v1/ping`). Optional `BELEGARCHIV_REMOTE_PROXY_SECRET` für HMAC-Header (`core/src/lib/remoteModuleProxy.js`). Weitere Proxys (Webhooks, mTLS) können darauf aufbauen.

## Cursor / KI

Projektregeln: `.cursor/rules/belegarchiv-apinterface-parity.mdc` (`alwaysApply`).

## Lokaler Workspace (dieser Ordner)

`core/`, `platform/`, `modules/uid/` und ggf. `modules/example-app/` sind **eigene Git-Repositories** (jeweils `origin` → GitHub). Das **Meta-Repo** liegt in `c:\Belegarchiv.ch` (nur README, `.cursor/`, `.gitmodules.example`, `.gitignore`) — `git add .` im Meta-Root vermeiden.

**Submodule im Meta-Repo** (`git submodule add …`) erst sinnvoll in einem **frischen Meta-Clone**, wenn die Pfade `core/` / `platform/` noch leer sind — sonst mit bestehenden Ordnern kollidieren. Bis dahin: so wie jetzt mit separaten Remotes in den Unterordnern arbeiten.

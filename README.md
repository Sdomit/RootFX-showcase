# RootFX

Read-only media library browser and indexer for VFX pipelines. Scan network storage, search across hundreds of thousands of assets, preview footage and image sequences, and send items directly into Nuke — all without moving or touching source media.

> **Private source — this repository is a project showcase.**

---

## What It Does

VFX studios accumulate enormous libraries on shared network storage. Finding the right footage usually means navigating deep folder trees through Windows Explorer or a slow generic DAM. RootFX replaces that with a fast, search-first browser purpose-built for the pipeline:

- **Full-text + metadata search** powered by Meilisearch — filter by type, extension, resolution, date
- **Proxied previews** — thumbnails and scrub previews generated in the background without touching source files
- **Nuke integration** — search inside Nuke, right-click to import, sets `file`, `first`, `last`, `colorspace` automatically
- **Admin console** — manage library roots, settings, and cache from a web UI with draft/validate/apply workflow
- **Cross-platform roots** — single config supports Linux, macOS, and Windows mapped/UNC paths for the same storage

---

## Architecture

```
Linux / macOS Server                     Windows Client
─────────────────                        ──────────────
FastAPI Backend  ◄────── HTTP ──────────► React Web UI
  │ SQLite (catalog)                      (search, browse, preview)
  │ Meilisearch (search index)
  │ ffmpeg / ffprobe (metadata + proxy)       │
  │ OpenImageIO (EXR / image metadata)        ▼
  │                                       Windows Bridge (FastAPI, localhost)
  └── SSH / UNC ──────────────────────►   Avalonia Launcher (desktop shell)
                                              │
                                              ▼
                                         Nuke Plugin
                                         (in-app browser + import)
```

**Data never moves** — the backend reads source media in place, generates proxies into a local `cache/` folder, and serves everything through controlled API endpoints. Library roots are read-only by design.

---

## Components

### Backend (`app/`)
FastAPI application running on Linux/macOS. Handles indexing, search, metadata extraction, proxy generation, and all media serving.

| Module | Responsibility |
|---|---|
| `app/core/` | Settings, models, path security, platform detection, root mapping |
| `app/indexer/` | File scanner, sequence grouper, version-based index management |
| `app/media/` | Thumbnail + preview generation (ffmpeg/OIIO), cache management |
| `app/search/` | Meilisearch integration, hybrid search, operator parsing |
| `app/profiles/` | Named filter profiles and synonym management |
| `app/admin/` | Draft/validate/apply config workflow, Meilisearch + bridge installer |
| `app/api/` | REST routes — search, item, thumb, preview, reindex, jobs |

### Web UI (`web/`)
React + Vite SPA. Asset grid with filter sidebar, detail viewer, admin console.

- Live search with operator support (`type:exr`, `res:4k`, `date:2024-01-01..2024-12-31`, `ext:mov`)
- Card view with cached thumbnail + alpha-aware hover preview
- Detail panel with metadata readout, proxy preview, and import action
- Admin console — edit draft config, validate, apply atomically, roll back from history

### Windows Bridge (`bridge/`)
Minimal FastAPI service running on Windows (localhost-only). Proxies actions that require Windows APIs:

- `POST /open-folder` — opens path in Explorer
- `POST /open-file` — opens file with default Windows handler
- `POST /open-mpv` — launches mpv for video playback
- `POST /import/nuke` — forwards import payload to the Nuke listener

All paths are validated against the configured mapped/UNC root allowlist.

### Avalonia Launcher (`desktop/RootFX.Launcher.Avalonia/`)
Windows desktop shell (C# / .NET 8 / Avalonia). Manages backend and bridge process lifecycle from a native Windows app — start, stop, health monitoring, log streaming — without requiring a terminal.

### Nuke Plugin (`plugins/nuke/`)
In-Nuke browser panel with search bar, thumbnail grid, and frame-count badges.

- Right-click → **Import to Nuke** creates a `Read` node with `file`, `first`, `last`, `origfirst`, `origlast`, `colorspace`
- Right-click → **Load RootFX Preview** loads proxy into Nuke viewer
- Right-click → **Open in Browser** jumps to item in the web UI
- Localhost listener accepts `POST /import` payloads from the web UI
- One-command installer for Windows (`.cmd`) and Python CLI

### Runtime CLI (`scripts/devctl.py`)
Development and operations controller:

```
devctl up              start backend (preflight checks → foreground uvicorn)
devctl doctor          preflight check — config, imports, roots, Meili, bridge
devctl reindex --full  full library reindex
devctl reindex --root  reindex one root
devctl verify          full verification: pytest + web build + live HTTP checks
```

---

## Features

### Indexing
- Parallel file scan with sequence grouping (image sequences collapsed to one item)
- Per-item metadata: resolution, frame range, duration, codec, colorspace, pixel format, file size, mtime, alpha detection
- Version-based index snapshots — activate/roll back any version via API
- Incremental reindex per root or per path prefix

### Search
- Meilisearch for full-text search with typo tolerance
- Falls back to SQLite if Meilisearch is unavailable (functional degraded mode)
- Operator syntax: `type:exr`, `ext:mov`, `res:4k`/`res:1080p`, `date:YYYY-MM-DD..YYYY-MM-DD`, `exact=true`
- Named profiles save filter combinations; synonym groups normalize terminology

### Preview Pipeline
- Thumbnails generated via ffmpeg or OIIO (EXR-aware)
- Proxy previews serialized to avoid memory spikes on large image sequences
- `cache_only=true` query parameter returns `409` immediately if proxy not cached (prevents UI from blocking on hover)
- `surface=detail|card` selects neutral detail preview vs alpha-aware card rendering
- Budget-limited cache per root with admin-triggered cleanup

### Admin Workflow
1. Open `/admin` → sign in
2. Edit settings in tabbed form (roots, server, indexing, cache, Meilisearch)
3. **Validate Draft** — runs schema + semantic + dependency checks
4. **Apply Draft** — atomically writes config, snapshots current, reloads services
5. **History / Rollback** — restore any previous snapshot

### Safety Model
- Library roots are read-only — no writes to source storage
- All filesystem access validated via resolved-path allowlist (symlink-safe)
- No shell-string execution — all ffprobe/ffmpeg calls use argv lists (`subprocess.run([...])`)
- Writes restricted to `config/`, `db/`, `cache/`, `indexes/`, `logs/`, `web_build/`
- Admin endpoints require token + localhost restriction
- Bridge rejects any path outside configured mapped/UNC roots

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.10+ / FastAPI / Uvicorn |
| Database | SQLite (catalog, job state) |
| Search | Meilisearch (full-text + filtered) |
| Media | ffmpeg / ffprobe, OpenImageIO (EXR) |
| Web UI | React + Vite |
| Windows shell | C# / .NET 8 / Avalonia |
| Windows bridge | FastAPI (localhost-only) |
| Nuke plugin | Python (PySide2/Qt) |
| Tests | pytest |

---

## Key Engineering Decisions

**Why a version-based index instead of in-place updates?**
Reindexing a large library takes time. Writing to a new version folder means the active index stays live and queryable until the new version is fully built and explicitly activated. No downtime, no partial-index states.

**Why serialize proxy generation?**
Generating previews for EXR image sequences is memory-intensive. Allowing concurrent generation on hover would spike memory use unpredictably on large frames. A serialized queue means predictable resource use at the cost of queue latency — acceptable for a browse workflow.

**Why a separate Windows Bridge?**
The backend runs on Linux/macOS. Actions like opening Explorer, launching mpv, or communicating with the Nuke listener are Windows-only. A localhost-only FastAPI bridge on Windows keeps the Linux backend clean and allows the web UI to trigger Windows actions without the backend needing to know about Windows APIs.

**Why draft/validate/apply for config changes?**
Live config edits on a running media server can break active sessions. The draft workflow means every change is validated against schema + semantic rules + dependency checks before being applied atomically. Snapshots make any bad apply instantly reversible.

**Why argv lists instead of shell strings for subprocesses?**
Shell-string construction for ffmpeg commands with user-supplied paths is a command-injection risk. All subprocess calls use argv lists — no shell interpretation, no escaping bugs, no injection surface.

---

## Status

Active development. Private source — this repository is a project showcase.

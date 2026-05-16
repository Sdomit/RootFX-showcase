<div align="center">

# RootFX

**Read-only media library browser and indexer for VFX pipelines — search, preview, and send to Nuke without touching source media.**

[![Stack: Python / FastAPI / React / Meilisearch](https://img.shields.io/badge/Stack-FastAPI%20%2F%20React%20%2F%20Meilisearch-009688?style=flat-square)]()
[![Platform: Linux + Windows](https://img.shields.io/badge/Platform-Linux%20%2B%20Windows-lightgrey?style=flat-square)]()
[![Showcase](https://img.shields.io/badge/Source-Private%20Showcase-lightgrey?style=flat-square)]()

> Private source — this repository is a project showcase.

</div>

---

## The Problem It Solves

VFX studios accumulate enormous libraries on shared network storage. Finding the right footage usually means navigating deep folder trees through Windows Explorer or a slow generic DAM.

RootFX replaces that with a fast, search-first browser purpose-built for the pipeline — with a Nuke plugin, proxied previews, and an admin console, all without moving or touching source media.

---

## Architecture

```
Linux / macOS Server                     Windows Client
─────────────────                        ──────────────
FastAPI Backend  ◄────── HTTP ──────────► React Web UI
  SQLite (catalog)                        (search, browse, preview)
  Meilisearch (search index)
  ffmpeg / ffprobe (metadata + proxy)        │
  OpenImageIO (EXR / image metadata)         ▼
                                         Windows Bridge (FastAPI, localhost)
  SSH / UNC ──────────────────────────►  Avalonia Launcher (desktop shell)
                                              │
                                              ▼
                                         Nuke Plugin
                                         (in-app browser + import)
```

**Data never moves** — the backend reads source media in place, generates proxies into a local `cache/` folder, and serves everything through controlled API endpoints. Library roots are read-only by design.

---

## Components

### Backend (`app/`)
FastAPI on Linux/macOS. Handles indexing, search, metadata extraction, proxy generation, and all media serving.

| Module | Responsibility |
|:---|:---|
| `app/core/` | Settings, models, path security, platform detection, root mapping |
| `app/indexer/` | File scanner, sequence grouper, version-based index management |
| `app/media/` | Thumbnail + preview generation (ffmpeg / OIIO), cache management |
| `app/search/` | Meilisearch integration, hybrid search, operator parsing |
| `app/profiles/` | Named filter profiles and synonym management |
| `app/admin/` | Draft/validate/apply config workflow, Meilisearch + bridge installer |
| `app/api/` | REST routes — search, item, thumb, preview, reindex, jobs |

### Web UI (`web/`)
React + Vite SPA. Asset grid with filter sidebar, detail viewer, and admin console.

- Live search with operator support: `type:exr`, `res:4k`, `date:2024-01-01..2024-12-31`, `ext:mov`
- Card view with thumbnail + alpha-aware hover preview
- Detail panel with metadata, proxy preview, and import action
- Admin console — edit draft config, validate, apply atomically, roll back from history

### Windows Bridge (`bridge/`)
Minimal FastAPI service (localhost-only) for Windows-specific actions:
- Open path in Explorer
- Open file with default handler
- Launch mpv for video playback
- Forward import payload to the Nuke listener

### Avalonia Launcher
Windows desktop shell (C# / .NET 8 / Avalonia). Manages backend and bridge process lifecycle from a native Windows app — start, stop, health monitoring, log streaming.

### Nuke Plugin (`plugins/nuke/`)
In-Nuke browser panel with search bar, thumbnail grid, and frame-count badges.

- Right-click **Import to Nuke** — creates a `Read` node with `file`, `first`, `last`, `colorspace` set automatically
- Right-click **Load RootFX Preview** — loads proxy into Nuke viewer
- Right-click **Open in Browser** — jumps to item in the web UI

---

## Key Features

### Indexing
- Parallel file scan with sequence grouping (EXR sequences collapsed to one item)
- Per-item metadata: resolution, frame range, duration, codec, colorspace, pixel format, file size, alpha detection
- Version-based index snapshots — activate or roll back any version via API
- Incremental reindex per root or per path prefix

### Search
- Meilisearch full-text with typo tolerance
- Operator syntax: `type:exr`, `ext:mov`, `res:4k`, `date:YYYY-MM-DD..YYYY-MM-DD`, `exact=true`
- Falls back to SQLite if Meilisearch is unavailable
- Named profiles save filter combinations; synonym groups normalize terminology

### Admin Workflow
1. Open `/admin` and sign in
2. Edit settings in tabbed form (roots, server, indexing, cache, Meilisearch)
3. **Validate Draft** — schema + semantic + dependency checks
4. **Apply Draft** — atomically writes config, snapshots current, reloads services
5. **History / Rollback** — restore any previous snapshot

### Safety
- Library roots are read-only — no writes to source storage
- All filesystem access validated via resolved-path allowlist (symlink-safe)
- No shell-string execution — all ffprobe/ffmpeg calls use argv lists
- Admin endpoints require token + localhost restriction

---

## Tech Stack

| Layer | Technology |
|:---|:---|
| Backend | Python 3.10+ / FastAPI / Uvicorn |
| Database | SQLite (catalog, job state) |
| Search | Meilisearch |
| Media | ffmpeg / ffprobe, OpenImageIO |
| Web UI | React + Vite |
| Windows shell | C# / .NET 8 / Avalonia |
| Windows bridge | FastAPI (localhost-only) |
| Nuke plugin | Python (PySide2/Qt) |
| Tests | pytest |

---

## Status

Active development. Private source — this repository is a project showcase.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Minutes" — a single-page, offline-first meeting notes app. The entire app is a static site: `index.html` (UI + all JS inline in one `<script>`) and `sw.js` (service worker). No build step, no dependencies, no package manager. Deployed to GitHub Pages from `main` via `.github/workflows/static.yml` (uploads the whole repo as the Pages artifact).

## Running locally

Serve the directory over HTTP (the service worker requires a real origin, `file://` will not work):

```
python -m http.server 3000
```

Then open http://localhost:3000. The same command is wired up in `.claude/launch.json`.

When iterating on `sw.js`, bump the `CACHE` constant (currently `'minutes-v1'`) or unregister the SW in DevTools — otherwise the cached old version will keep serving.

## Architecture

Everything app-related lives inside the IIFE at the bottom of `index.html` (starting around line 1201). Section banners (`// ── DATA STORE ──`, `// ── PARSE ──`, etc.) mark the major regions.

### Data model

Single root object persisted under one IndexedDB key (`minutes_db` / store `kv` / key `minutes_v1`):

```
{ meetings: { [id]: Meeting }, actions: Action[], decisions: Decision[] }
```

`actions` and `decisions` are **denormalized indexes** — they are not the source of truth. The source of truth is each meeting's `notes` text. On every save, `indexMeeting()` re-parses that meeting's notes and rewrites its slice of the global `actions`/`decisions` arrays. This lets the Actions and Tags consolidated views render without re-scanning every meeting. When changing note schema or parsing rules, update `indexMeeting` / `clearIndex` accordingly so the indexes stay consistent.

On first load the async init in the IIFE migrates any legacy `localStorage` payload into IndexedDB.

### Notes syntax (parsed live)

Plain-text conventions in the notes editor that the parsers in `parseActions`, `parseDecisions`, `parseTags` recognize:

- `@name task text` → action item assigned to `name` (multiple `@`s = multiple owners)
- `→ text` or `** text` → decision
- `!YYYY-MM-DD`, `!today`, `!tomorrow`, `!nextweek` → deadline on an action (expanded by `expandDeadlineShortcuts`)
- `#tag` → tag

The contenteditable layer round-trips through `textToHTML` / `htmlToText` so indent/outdent and list semantics are preserved as plain text on disk.

### UI structure

Three tabs in the left rail driven by `switchTab(tab)`:

- **Notes** — note list + editor in `<main>`.
- **Actions** — `renderActionsByPerson` (sidebar grouping) and `renderActionsForPerson` / `renderAllActions` (table injected into `<main>` via `#actions-table-container`).
- **Tags** — `renderTagsPanel` + `showTagNotes` (table injected via `#tags-table-container`).

`switchTab` is responsible for showing/hiding the editor, empty state, and the dynamically inserted table containers — when adding a new tab or main-area view, mirror that show/hide bookkeeping or stale panels will leak across tabs.

### Persistence + dirty tracking

Two debounced savers (`scheduleSave` + `saveNote` for note body, `saveMeta` for title/date/time/category) write the whole root object back to IndexedDB. `setDirty` / `updateStatusBar` drive the status dot. `window.app` exposes the handful of functions referenced by inline `onclick=` attributes in the HTML — anything called from markup must be re-exported there.

### Offline / service worker

`sw.js` is a cache-first SW: same-origin assets are cached on first fetch; Google Fonts use a stale-while-revalidate variant. The install step pre-caches `./` only, so other assets populate on demand.

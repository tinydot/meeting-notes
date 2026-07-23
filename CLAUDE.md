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

When iterating on `sw.js` **or shipping any change to `index.html`**, bump the `CACHE` constant in `sw.js` — the SW serves same-origin assets cache-first, so without a bump the cached old version keeps serving.

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

The contenteditable layer round-trips through `textToHTML` / `htmlToText`: notes are a flat numbered list, one line per `<li>`, preserved as plain text on disk. `textToHTML` runs each line through `renderNoteLine`, which escapes it and wraps `#tag` tokens in `<span class="tag-link">` and `@mention` tokens in `<span class="person-link">`; `htmlToText` reads each span's `textContent`, so the round-trip stays lossless. A single document-level click handler opens the cross-reference modal for whichever link was clicked — `.tag-link` → `openTagModal`, `.person-link` → `openPersonModal` — wherever those links appear (editor, inferred panel, backlinks).

### UI structure

Three tabs in the left rail driven by `switchTab(tab)`:

- **Notes** — note list + editor in `<main>`.
- **Actions** — `actionsView.mode` selects the `<main>` view (all injected into `#actions-table-container`): `review` (default) → `renderReview`, `all` → `renderAllActions`, `detail` → `renderActionsForPerson`. `renderActionsByPerson` draws the sidebar, which always leads with a **Review** inbox + **All actions** nav. The Review inbox (`reviewItems` buckets open actions into Overdue / Due this week / No deadline) offers one-tap Done / Snooze 1w / Reschedule (`markDone` / `reviewSnooze` / `reviewReschedule`), each a line rewrite via `rewriteActionLine`. `markDone` is the single mark-done flow shared by the inferred panel, the action tables, and Review; every mark-done button carries `data-mid` / `data-idx` / `data-full`, and `rewriteActionLine` locates the target line by the action's recorded line index (re-verified by re-parsing) rather than substring matching. `updateReviewBadge` keeps the count pill on the Actions rail-tab in sync.
- **Tags** — `renderTagsPanel` + `showTagNotes` (table injected via `#tags-table-container`).

**Cross-reference modal (`#tag-modal`).** A shared popup that traces a thread without leaving the editor. `openRefModal({title, entries, highlightRe, empty, unit})` splits entries into two ordered groups — open items with a deadline (soonest first), then everything without a deadline or already completed (newest meeting first) — and paints them via `refItemHTML`; rows call `openRefNote(id)` to jump to the source note. Two collectors feed it: `tagEntries(tag)` (every note line carrying a `#tag`) backs `openTagModal`, and `personEntries(person)` (the denormalized `data.actions` index, filtered by owner) backs `openPersonModal`.

**Backlinks ("Related threads").** `renderBacklinks` adds a third section to the inferred panel under the editor: for each tag on the current note (read live from the editor), it lists other meetings sharing that tag. `updateInferred` calls it and folds its return into the panel's show/hide condition. Tag chips there reuse `.tag-link` (so they open the modal); meeting names call `openNote(id, true)`.

`switchTab` is responsible for showing/hiding the editor, empty state, and the dynamically inserted table containers — when adding a new tab or main-area view, mirror that show/hide bookkeeping or stale panels will leak across tabs.

### Persistence + dirty tracking

Two debounced savers (`scheduleSave` + `saveNote` for note body, `saveMeta` for title/date/time) write the whole root object back to IndexedDB. `setDirty` / `updateStatusBar` drive the status dot. `window.app` exposes the handful of functions referenced by inline `onclick=` attributes in the HTML — anything called from markup must be re-exported there.

### Google Drive backup

The `// ── GOOGLE DRIVE BACKUP ──` section backs the whole `data` root up to a single `minutes-backup.json` in the user's Drive. It's a pure browser OAuth flow (Google Identity Services token client, no server, no client secret) scoped to `drive.file` — the app can only touch the one file it creates. Because there's no server, each deployer supplies **their own** OAuth client ID (registered for their Pages origin under "Authorized JavaScript origins"); it's entered in the `#backup-modal` and persisted, along with `driveFileId`, `lastBackup`, and the `autoBackup` toggle, to `localStorage` under `minutes_settings` (`loadSettings`/`saveSettings`). The access token lives in memory only — never persisted. The GIS script is injected lazily on first use (`loadGis`) so offline launches stay clean, and because it and the Drive API are cross-origin the same-origin SW leaves them to the network (backup simply no-ops offline). `backupNow` multipart-creates or PATCH-updates the file (recovering a lost `driveFileId` via `findBackupFile`); `restoreNow` downloads it and runs it through `applyImportedData` — the shared payload-swap helper also used by file import. When `autoBackup` is on, `saveData` triggers a debounced, silent `scheduleAutoBackup` (never prompts).

### Offline / service worker

`sw.js` is a cache-first SW: same-origin assets are cached on first fetch; Google Fonts use a stale-while-revalidate variant. The install step pre-caches `./` only, so other assets populate on demand.

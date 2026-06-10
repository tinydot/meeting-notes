# Minutes

A tiny, offline-first meeting notes app. One HTML file, no build step, no dependencies, no backend. Everything lives in your browser's IndexedDB.

## Features

- **Local-only.** Notes are stored in IndexedDB on your device. Nothing is sent anywhere.
- **Works offline.** A service worker caches the app shell so it loads without a network.
- **Plain-text shortcuts** that get parsed into structured views:
  - `@name task text` — action item (multiple `@`s = multiple owners)
  - `→ text` or `** text` — decision
  - `!YYYY-MM-DD`, `!today`, `!tomorrow`, `!nextweek` — deadline on an action
  - `#tag` — tag
- **Review inbox** that surfaces open action items by urgency (overdue / due this week / no deadline) with one-tap Done, Snooze, and Reschedule, plus a count badge on the Actions tab.
- **Consolidated views** for Actions (grouped by person, with overdue counts) and Tags, derived live from your notes.
- **Import / export** as JSON for backup or moving between devices.

## Running locally

A service worker requires a real HTTP origin, so opening `index.html` directly via `file://` won't fully work. Serve the directory instead:

```
python -m http.server 3000
```

Then visit http://localhost:3000.

## Deployment

Pushes to `main` are deployed to GitHub Pages by `.github/workflows/static.yml`, which uploads the entire repository as the Pages artifact.

## Editor

- The notes editor is a contenteditable numbered list that round-trips through plain text, so what you see is what gets saved.

## Files

- `index.html` — the entire app (markup, styles, and JS in one file).
- `sw.js` — service worker for offline caching.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Langue de communication

Toujours communiquer avec l'utilisateur **en français**, quelle que soit la langue utilisée dans les questions ou les fichiers.

## Project overview

Fermapro is a suite of standalone French-language HTML applications for a windows & closures company (menuiseries & fermetures) based in Fondettes (37, Indre-et-Loire). There is no build system, no package manager, and no framework — each file is a self-contained HTML/CSS/JS app deployed as static files via GitHub Pages.

## Architecture

### No build step
Open HTML files directly in a browser. Nothing to install or compile. Changes are visible immediately on reload.

### Apps in this repository

| File | Purpose |
|---|---|
| `fermapro-stock.html` | Main stock management PWA — the core app |
| `agent-notes-devis.html` | Claude AI agent that generates explanatory notes for customer quotes |
| `assistant-taches.html` | Task management for office assistants |
| `direction-taches.html` | Task management for management |
| `google-business-generator.html` | Claude AI agent for generating Google Business posts |
| `manifest.json` | PWA manifest (points to `fermapro-stock.html` as start URL) |

### Backend: Google Apps Script

All persistent data lives in **Google Sheets** accessed via a deployed Google Apps Script web app. The URL is stored in `localStorage` under `fp_script_url` and defaults to `DEFAULT_SCRIPT_URL` (defined near the top of the JS in `fermapro-stock.html`).

**API pattern** (in `fermapro-stock.html`):
- `apiGet(action)` — `fetch(SCRIPT_URL + '?action=<action>&t=<timestamp>')`
- `apiPost(body)` — `fetch(SCRIPT_URL, { method:'POST', body: JSON.stringify(body) })`

Actions include: `getPieces`, `getMouvements`, `getUsers`, `getDepots`, `savePiece`, `addMouvement`, `saveUser`, `deleteUser`, `deletePiece`, `saveDepots`, `sendEmailAlert`.

### localStorage keys (fermapro-stock.html)

| Key | Contents |
|---|---|
| `fp_script_url` | Configured Apps Script URL |
| `fp_users` | Cached user list (fallback when offline) |
| `fp_last_user` | ID of last logged-in user |
| `fp_connexions` | Local login audit log (never synced to server) |
| `fp_email_cfg` | Email alert configuration |

### Authentication

PIN-based, client-side only. Users pick their profile from a grid, enter a 4-digit PIN. The PIN is stored with the user record in Google Sheets and cached in localStorage. Sessions are not server-validated — this is a trusted-team tool, not a security boundary.

### Role / permission system

Roles: `admin`, `tech`, `commercial`, `assistante`, `custom`. Each user has boolean permission flags: `canView`, `canMove`, `canEdit`, `canPrint`, `canImport`, `canAdmin`. The UI hides/shows elements with the CSS classes `.admin-only`, `.nav-admin`, `.print-only`, `.import-only` based on these flags set at login.

### Multi-depot stock

Each piece (`piece`) has a `stocks` field stored as a JSON string (keyed by depot ID). Stock totals are computed client-side by summing across depots or filtering by the active depot tab.

### Claude API apps (agent-notes-devis.html, google-business-generator.html)

These call the Anthropic API **directly from the browser** using `anthropic-dangerous-direct-browser-access: true`. The API key is entered by the user and stored in `localStorage` (`fermapro_agent_apikey`). They use `claude-sonnet-4-20250514` with the `web_search_20250305` tool. File inputs are base64-encoded before being sent as `document` or `image` content blocks. Images are compressed client-side to stay under the 4 MB limit.

### External CDN libraries (fermapro-stock.html)

- **SheetJS / xlsx 0.18.5** — Excel import/export
- **html5-qrcode 2.3.8** — QR code scanning via device camera
- **Chart.js 4.4.1** — Bar/doughnut charts on dashboard and connexions pages

### CSS design system

CSS custom properties are defined in `:root` at the top of each file. Core palette shared across apps:
- `--navy` / `--navy2` / `--navy3`: primary brand color (#2D3580)
- `--pink` / `--pink2`: accent (#E8457A)
- `--bg`, `--surface`, `--border`: layout neutrals
- `--green`, `--amber`, `--red`: stock status colors
- Fonts: Nunito (headings `--fn`), Open Sans (body `--fb`), JetBrains Mono (codes `--fm`)

### SPA page routing (fermapro-stock.html)

No URL routing. Pages are `<div class="page" id="page-*">` elements that get the `.active` class added/removed by `goPage(name, navItem)`. The active nav item gets the `.active` class.

## Key conventions

- All user-visible text is in French.
- Dates/times are formatted with `toLocaleDateString('fr-FR')` / `toLocaleString('fr-FR')`.
- Toast notifications (`toast(msg, type)`) are used for all user feedback — `type` is `''` (default navy), `'err'`, or `'warn'`.
- The sync status bar (`.sync-bar`) reflects connection state: `ok`, `syncing`, `error`, `config`.
- `loadAll()` is the central data-refresh function; it fetches pieces and mouvements in parallel, then re-renders all active views.
- When adding a new action that writes to the backend, call `loadAll()` after `apiPost()` resolves to keep the UI in sync.
- Pieces are identified by `ref` (the stock reference code, e.g. `FP-0001`). QR codes encode this ref.
- Stock movements (`mouvement`) always record: `ref`, `nom`, `type` (`entrée`/`sortie`), `qty`, `depotId`, `raison`, `user` (name), `userId`, `stockApres`, `date`.

## Deployment

The app is deployed via GitHub Pages from the `main` branch at `https://fermapro.github.io/fermapro/`. No CI pipeline exists — pushing to `main` deploys automatically via GitHub Pages.

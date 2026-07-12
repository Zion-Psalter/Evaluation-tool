# Zion Psalter — Evaluation Form

Custom replacement for the Fillout embed that used to live at `singzion.com/evaluate`. Built so the "Title || Artist" picker reads live from the Zion Psalms Google Sheet instead of a manually-maintained, 1000-option-capped dropdown.

If you're picking this project up for the first time, read this whole file before changing anything — it covers the architecture, where things live, and how to make common changes safely.

---

## 1. How it fits together

```
┌─────────────────────┐        GET  ?action=catalog        ┌──────────────────────────┐
│                      │ ───────────────────────────────►  │                          │
│   index.html         │                                     │   Apps Script Web App    │
│   (static page,      │        POST  (form submission)      │   (Code.gs, bound to     │
│   hosted on          │ ───────────────────────────────►    │   the Zion Psalms sheet) │
│   GitHub Pages)       │  ◄───────────────────────────────  │                          │
│                      │        JSON { ok, ... }             └────────────┬─────────────┘
└──────────┬───────────┘                                                  │
           │                                                    reads/writes
           │ embedded via                                                  │
           │ Google Sites "Embed" block                        ┌───────────▼─────────────┐
           ▼                                                    │   Zion Psalms            │
┌──────────────────────┐                                        │   Google Sheet           │
│  singzion.com/evaluate│                                        │   - Sheet 1: catalog     │
│  (Google Site)         │                                        │   - "Evaluation          │
└──────────────────────┘                                        │     Responses" tab       │
                                                                  └──────────────────────────┘
```

Two independently-deployed pieces:

| Piece | File(s) | Where it runs | What it does |
|---|---|---|---|
| Frontend | `index.html` | GitHub Pages (static hosting, no build step) | Renders the form, fetches the live catalog for autocomplete, POSTs submissions |
| Backend | `Code.gs`, `appsscript.json` | Google Apps Script, bound to the Zion Psalms sheet | Serves catalog data as JSON; validates and writes submissions to a "Evaluation Responses" tab; saves chart uploads to Drive |

There is no database and no third-party hosting account beyond GitHub — the Google Sheet *is* the database.

## 2. Repo layout

```
your-repo/
├── index.html          ← site root, this is what GitHub Pages serves
├── README.md            ← this file
└── apps-script/
    ├── Code.gs
    └── appsscript.json
```

`apps-script/` is here for version control only — Apps Script doesn't deploy from GitHub automatically. You paste `Code.gs` / `appsscript.json` into the Apps Script editor by hand (see §4). If someone sets up [`clasp`](https://github.com/google/clasp) later, this repo is already laid out to support that.

## 3. Who needs access to what

- **Google Sheet ("Zion Psalms")** — whoever maintains `Code.gs` needs edit access, since the script is bound to and executes as a user with access to this sheet.
- **GitHub repo** — anyone editing `index.html` or the styling.
- **Google Sites (singzion.com)** — whoever needs to update the embed block if the GitHub Pages URL ever changes.
- **Apps Script deployment** — only the person who owns the deployment can push new versions (Deploy → Manage deployments). If that person leaves the project, redeploy under a new account and update `CONFIG.API_URL` in `index.html`.

## 4. Deploying the backend (Apps Script)

1. Open the **Zion Psalms** Google Sheet.
2. **Extensions → Apps Script**.
3. Paste in `Code.gs` (replacing the default boilerplate).
4. Project Settings (gear icon) → check "Show `appsscript.json`" → paste in `appsscript.json`.
5. **Deploy → New deployment → Web app**:
   - Execute as: **Me**
   - Who has access: **Anyone**
   - Authorize the requested permissions (reads/writes the spreadsheet, creates files in Drive for chart uploads).
6. Copy the **Web app URL** (ends in `/exec`) — this goes into `index.html`.

**Every time you edit `Code.gs`**, you must go to **Deploy → Manage deployments → pencil icon → New version** for the change to go live at the same URL. Just saving the script is not enough.

Key constants at the top of `Code.gs` you may need to touch:

```js
var CATALOG_SHEET_INDEX = 0;              // which tab is the song catalog (0 = first tab)
var RESPONSES_SHEET_NAME = 'Evaluation Responses';
var TRACK_COL_NAME = 'Track Name';
var ARTIST_COL_NAME = 'Artist Name(s)';
var COMBINED_COL_NAME = 'Track || Artist';
var RATE_LIMIT_SECONDS = 30;
var MAX_UPLOAD_BYTES = 10 * 1024 * 1024;  // 10 MB
```

## 5. Deploying the frontend (GitHub Pages)

1. In `index.html`, find near the top of the `<script>` block:
   ```js
   var CONFIG = {
     API_URL: 'PASTE_YOUR_APPS_SCRIPT_WEB_APP_URL_HERE'
   };
   ```
   Set it to the `/exec` URL from step 4.6.
2. Push `index.html` to the repo (root, or `main` branch).
3. **Repo Settings → Pages** → Source: Deploy from branch → `main` / `(root)`.
4. GitHub gives you a URL like `https://<org-or-user>.github.io/<repo>/`. Open it, confirm the autocomplete populates and a real test submission lands in the "Evaluation Responses" tab.

### Previewing locally before pushing

`index.html` is a single static file with no build step — just open it directly in a browser, or serve it locally:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000/index.html
```

As long as `CONFIG.API_URL` points at a live Apps Script deployment, local previews will pull real catalog data and can submit real rows — be careful about test-submitting into the live sheet. Consider a second Apps Script deployment pointed at a duplicate sheet if you need a true sandbox.

## 6. Embedding in Google Sites

1. Edit the **Evaluation Portal** page on singzion.com.
2. Remove the old Fillout embed block (if not already removed).
3. Insert → **Embed** → paste the GitHub Pages URL.
4. Publish.

## 7. Common changes

**Add or remove a form field**
Edit the relevant `<div class="field">` block in `index.html`, then add/remove the matching key in the `payload` object inside the `submit` handler (near the bottom of the `<script>` block), then add/remove the matching column in `RESPONSE_HEADERS` and the `sheet.appendRow([...])` call in `Code.gs`. Keep the three arrays (HTML field, JS payload key, sheet column) in sync — nothing enforces this automatically.

**Change the color palette or fonts**
Everything is driven by CSS custom properties in the `:root` block at the top of `<style>` (`--navy-900`, `--gold`, `--font-display`, etc.). Change the values there rather than hunting for individual colors in the rest of the file. Current values match the brand tokens in this project's `Brand.md` and the Song Database browser (`zion-psalms-browser_6.html`) — Fraunces for display type, Inter for body, same navy/gold/parchment palette.

**Change the rating scale (currently 0–10, hidden from the UI)**
Update `min`/`max` on the three `<input type="range">` elements, and the `RATE...` values are stored as-is in `Code.gs` — no other scaling logic exists. The UI intentionally never displays the numeric value (see `visibility: hidden` toggling in the slider JS) — if that should change, look for `valueLabel.style.visibility`.

**Point at a different catalog tab or column names**
Change `CATALOG_SHEET_INDEX`, `TRACK_COL_NAME`, `ARTIST_COL_NAME`, `COMBINED_COL_NAME` at the top of `Code.gs`, redeploy (§4).

**Adjust spam protection**
`RATE_LIMIT_SECONDS` in `Code.gs` controls the per-email cooldown. The honeypot field is the hidden `#f-website` input in `index.html` — don't remove it without removing the corresponding check in `Code.gs`'s `doPost`.

## 8. Testing checklist before relying on this in production

- [ ] Autocomplete loads and returns results for a few known tracks/artists, including ones that were previously excluded from the Fillout dropdown (e.g. Poor Bishop Hooper, Enter the Worship Circle).
- [ ] Submitting with all fields filled writes a correct row to "Evaluation Responses".
- [ ] Submitting with only the required fields (Track/Artist + 3 sliders) succeeds; leaving any of those blank is blocked client-side.
- [ ] A chart file upload produces a working, view-only Drive link in the response row.
- [ ] Submitting twice quickly from the same email is rate-limited (should show an error the second time).
- [ ] The embedded version inside Google Sites looks and behaves the same as the standalone GitHub Pages URL.

## 9. Known limitations

- Rate limiting/honeypot is basic spam deterrence, not real bot protection — fine for a low-traffic evaluation form, not bulletproof.
- Anonymous submissions (no email entered) share a single rate-limit bucket, so heavy anonymous traffic could throttle itself.
- No admin UI for reviewing/moderating submissions — that's just the "Evaluation Responses" sheet tab.
- The catalog is only as fresh as the sheet; there's no caching layer, so very large catalogs (many thousands of rows) could make `doGet` slower over time. Not a concern at current catalog size.

## 10. Project history / origin

This form replaced a Fillout embed that capped the Title || Artist dropdown at 1000 manually-entered options, which had already caused some artists to be excluded. This version reads the catalog live from the Zion Psalms sheet on every page load, so there's no manual list-maintenance step and no cap.

# SHV ⇄ Walmart Load Bridge

A single-page, no-build-step web app with two buttons:

- **Fetch Loads** — calls the Walmart Freight Tender API (`GET /api/sap/loads`), then automatically sanitizes every load.
- **Sanitize & Push** — pushes the sanitized, valid loads to the SHV Logistics SOR Integration API (`POST /api/sor/loads`).

Everything runs client-side in the browser — there's no backend/server, so it can be hosted directly on GitHub Pages.

## Sanitization rules applied

| Walmart field | → | SHV field | Transform |
|---|---|---|---|
| `load_no` | → | `load_number` | trimmed, validated against `LD-<digits>` |
| `frt_ord_no` | → | `bol_number` | trimmed |
| `shipper_nm` | → | `shipper_name` | trimmed |
| `orig_city` / `orig_st` | → | `origin_city` / `origin_state` | trimmed |
| `dest_city` / `dest_st` | → | `destination_city` / `destination_state` | trimmed |
| `shp_dt` (MMDDYYYY) | → | `ship_date` (DDMMYYYY) | day/month reordered |
| `del_dt` (MMDDYYYY) | → | `delivery_date` (DDMMYYYY) | day/month reordered |
| `wgt` (e.g. `"41,860 lbs"`) | → | `weight` | commas/units stripped → plain integer |
| `mode` | → | `equipment_type` | `AMBIENT` → `Dry Van 53'`, `FREEZER` → `Reefer 53'` |

Any load with a missing/unparseable field, an unrecognized `mode`, or a value that fails
SOR formatting rules is flagged **needs review** and is **not** pushed — it stays visible
in the table with the specific validation error, and the invalid load is simply excluded
from the push payload rather than sent with guessed data.

The authenticated account email (`abhijeet99sawant8@gmail.com`) is hardcoded in `index.html`
as the bearer token for both APIs, as required by the docs.

## Deploying to GitHub Pages

1. Create a new **public** GitHub repository, e.g. `shv-load-bridge`.
2. Push these two files (`index.html`, `README.md`) to the repo's default branch:
   ```bash
   git init
   git add index.html README.md
   git commit -m "SHV/Walmart load bridge"
   git branch -M main
   git remote add origin https://github.com/<your-username>/shv-load-bridge.git
   git push -u origin main
   ```
3. In the repo, go to **Settings → Pages**.
4. Under **Build and deployment**, set **Source** to `Deploy from a branch`, branch `main`, folder `/ (root)`.
5. Save. GitHub will publish the site at:
   `https://<your-username>.github.io/shv-load-bridge/`
6. Open that URL — the app works with no further setup (no npm install, no build).

## Deploying to Vercel

The included `vercel.json` tells Vercel this is a plain static site (clean URLs, and
`index.html` served with `no-cache` so you always get the latest deploy instead of a
stale cached copy).

**CLI:**
```bash
npm install -g vercel
vercel login
vercel --prod
```
Accept the defaults (no framework, no build command, output directory = root).

**Or via the dashboard:** import the repo (or drag-and-drop the folder) at
[vercel.com/new](https://vercel.com/new), leave the framework preset as "Other", and deploy —
`vercel.json` handles the rest automatically.

## Notes / things to watch for

- **CORS**: both API calls are made directly from the browser to the Walmart and SHV
  domains. If either portal doesn't send permissive CORS headers, the browser will block
  the request and the app will show a network error in the Activity Log — that's a
  server-side portal setting, not something fixable from this app's code.
- **Rate limits**: Walmart allows 60 GET/min; SHV allows 30 POST/min and 60 GET/min, and
  caps accounts at 50 stored loads. The app surfaces `429` responses with the
  `Retry-After` value in the Activity Log.
- **Idempotency**: SHV pushes are upserts keyed by `load_number`, so re-clicking
  **Sanitize & Push** is safe to retry after fixing a rejected load.

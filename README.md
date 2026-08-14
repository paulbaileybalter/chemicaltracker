# Balter Brewing — Chemical Tracker

A password-protected weekly chemical stocktake tool, replacing the old `Chemical_Tracker.xlsx`. Enter Friday morning's stock count and it works out what to order, groups it by supplier so you can raise a PO, and charts usage over time. Built the same way as the other Balter sites (13:30 Handover, Daily Packaging Handover, Logistics Daily Handover): a static site behind a Cloudflare Worker login gate, with multi-device sync proxied server-side through JSONBin.io so no secret ever reaches the browser.

## What it does

- **Friday stocktake** — one table of all 19 chemicals, grouped by supplier (Sopura / Nalco / EcoLab), matching the old spreadsheet's Template tab. Type in stock on hand; min stock, status, and a suggested order quantity are all shown alongside it.
- **Order to place** auto-fills from the suggested quantity but is fully editable per chemical — overtype it any week without breaking next week's suggestion.
- **Purchase orders** — chemicals needing stock are grouped into one card per supplier, with that supplier's delivery cadence and order contacts (pulled from the old spreadsheet's ordering notes) and a "Copy PO" button that copies a ready-to-paste order for that supplier.
- **Usage trends** — pick any chemical to see a chart of stock on hand over recent weeks, with a min-stock reference line, average weekly usage, and an estimated weeks-of-cover figure.
- **Stocktake history** — every past week's counts, who counted them, and how many items were below min stock.
- **Copy order summary** / **Download PDF** — same pattern as the other sites: one button copies the whole week's orders as email-ready text, the other opens a print-formatted version of the sheet for saving as a PDF.

The 15 most recent stocktakes from the old spreadsheet's Dashboard tab are pre-loaded, so the trend charts and history have real data from day one. Those imported weeks don't have order quantities on record (the old sheet didn't track that), so usage figures for them are estimated from the current min stock level — anything logged going forward is exact. This is flagged in the app itself.

## Repo structure

```
wrangler.jsonc       — Worker config (points at src/worker.js and public/)
package.json         — just the wrangler dev dependency
src/worker.js         — the entire server: auth gate + /api/sync proxy + static file fallback
public/               — the actual site (index.html, manifest.json, icons)
```

## One-time setup

### 1. Create a JSONBin.io bin

1. Sign up at [jsonbin.io](https://jsonbin.io) (free tier is plenty for this).
2. Get your **Master Key** (X-Master-Key) from the API Keys page.
3. Create a new bin with an empty JSON object as its content, e.g. `{}`. Copy its **Bin ID** from the URL or dashboard.

### 2. Push this repo to GitHub

Create a new GitHub repo and push these files to it (a private repo is recommended, though nothing sensitive lives in the code itself since secrets are set separately in Cloudflare).

### 3. Connect it to Cloudflare via Workers Builds (Git integration)

Drag-and-drop won't work here since a Worker script has to actually run — this needs the Git-connected deploy path:

1. In the Cloudflare dashboard: **Workers & Pages → Create → Workers Builds** (or **Connect to Git** if prompted from the Workers overview).
2. Pick the GitHub repo you just created.
3. Build settings: no build command needed — Wrangler picks up `wrangler.jsonc` automatically. Leave the root directory as `/`.
4. Deploy. The first deploy will fail health checks until secrets are set (next step) — that's expected.

### 4. Set the four secrets

In the Worker's **Settings → Variables and Secrets**, add these four as type **Secret** (not Text):

| Name | Value |
|---|---|
| `SITE_PASSWORD` | The shared password your team will type in to get past the login screen |
| `SESSION_SECRET` | A long random string (e.g. generate one with `openssl rand -base64 32`) — used to sign session cookies. Don't reuse this across the sibling sites. |
| `JSONBIN_BIN_ID` | The Bin ID from step 1 |
| `JSONBIN_API_KEY` | The X-Master-Key from step 1 |

After saving secrets, redeploy (or it may auto-redeploy) and the site should come up behind the login screen.

## Local development

```
npm install
cp .dev.vars.example .dev.vars   # then fill in real values
npm run dev
```

`.dev.vars` holds secrets for local `wrangler dev` only — it's gitignored, never commit it. Wrangler loads it automatically.

## Using it day to day

- **Sign in**: everyone uses the same `SITE_PASSWORD`. The session lasts 7 days per browser before it asks again.
- **Log out**: button in the top-right.
- **Enter Friday's count**: type stock on hand for each chemical. The status pill (OK / below min) and the suggested order quantity update as you type.
- **Adjust an order**: the "Order to place" box starts out matching the suggestion but you can type over it any time — e.g. to round up to a full pallet, or hold off ordering something you know is arriving another way.
- **Raise a PO**: open the Purchase Orders card, find the supplier, hit "Copy PO" and paste into an email — or use "Copy order summary" in the header for everything across all three suppliers at once.
- **Next week**: once Friday's counts are in, click "Next week" in the header. This archives the current stocktake into history (so it shows up in trends and history) and opens a fresh sheet dated the following Friday.
- **Notes**: the small text field on each chemical row is for anything worth flagging — e.g. why a chemical used more than expected that week — the same purpose as the "Comments/Reasoning for Chemical Loss" column in the old spreadsheet.
- **Everyone signed in shares the one tracker** — there's no per-room link, so whoever does the count, everyone else sees the same numbers.

## If something's not syncing

- Check the Worker's logs in the Cloudflare dashboard (Observability is enabled in `wrangler.jsonc`) — a 502 from `/api/sync` usually means the JSONBin Bin ID or API key secret is wrong.
- The site still falls back to saving locally on that device if `/api/sync` is unreachable, so nobody loses their in-progress count even if sync is temporarily down.

## Updating the chemical list

The chemical master list (name, supplier, min stock, storage) lives near the top of the `<script>` block in `public/index.html`, in the `CHEMICALS` array. Adding a new chemical, retiring one (set `active: false` like `Cellarwash`), or changing a min stock level is a matter of editing that array and redeploying — there's no separate database for it, since the list changes rarely compared to the weekly counts.

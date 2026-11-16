# Balter Brewing — Chemical Tracker

A password-protected weekly chemical stocktake tool, replacing the old `Chemical_Tracker.xlsx`. Enter Friday morning's stock count and it works out what to order, groups it by supplier so you can raise a PO, and charts usage over time. Built the same way as the other Balter sites (13:30 Handover, Daily Packaging Handover, Logistics Daily Handover): a static site behind a Cloudflare Worker login gate, with multi-device sync stored server-side in a Cloudflare KV namespace — everything stays inside Cloudflare's own network, nothing is sent to a third-party service.

## What it does

- **Friday stocktake** — one table of all 19 chemicals, grouped by supplier (Sopura / Nalco / EcoLab), matching the old spreadsheet's Template tab. Type in stock on hand; min stock, status, and a suggested order quantity are all shown alongside it.
- **Order to place** auto-fills from the suggested quantity but is fully editable per chemical — overtype it any week without breaking next week's suggestion.
- **Purchase orders** — chemicals needing stock are grouped into one card per supplier, with that supplier's delivery cadence and order contacts (pulled from the old spreadsheet's ordering notes) and a "Copy PO" button that copies a ready-to-paste order for that supplier.
- **Usage trends** — pick any chemical to see a chart of stock on hand over recent weeks, with a min-stock reference line, average weekly usage, and an estimated weeks-of-cover figure.
- **Year dashboard** — click "Year dashboard" in the header to see every chemical plotted together on one chart, with a timescale picker (8 / 12 / 26 / 52 weeks / all time) that controls both the combined chart and the mini-multiples grid below it — your choice is remembered on that device. Most chemicals are normalized to % of their own min stock (100% is the reorder line). Chemicals already tracked as a raw % full (Synergex 1000L, ACID700, ALKS500) are shown as dashed lines with their actual reading instead, since dividing an already-low reorder point (e.g. 30%) by itself would blow the scale out even at a normal reading. Also includes a small trend card for every chemical so you can spot outliers at a glance. It's a real link (`#dashboard`), so it's bookmarkable and the browser back button works.
- **Stocktake history** — every past week's counts, who counted them, and how many items were below min stock. Click any row to open it and adjust counts, order quantities, or notes — handy when an order changes after the fact. This edits that week's record directly and doesn't touch the current stocktake.
- **Copy order summary** / **Download PDF** — same pattern as the other sites: one button copies the whole week's orders as email-ready text, the other opens a print-formatted version of the sheet for saving as a PDF.

The 15 most recent stocktakes from the old spreadsheet's Dashboard tab are pre-loaded, so the trend charts and history have real data from day one. Those imported weeks don't have order quantities on record (the old sheet didn't track that), so usage figures for them are estimated from the current min stock level — anything logged going forward is exact. This is flagged in the app itself.

## Repo structure

```
wrangler.jsonc       — Worker config (points at src/worker.js and public/)
package.json         — just the wrangler dev dependency
src/worker.js         — the entire server: auth gate + /api/sync proxy + static file fallback
public/               — the actual site (index.html, manifest.json, favicon.svg, icons)
```

## One-time setup

### 1. The KV namespace sets itself up

`wrangler.jsonc` already declares a `CHEM_KV` binding with no `id` — on your first `npx wrangler deploy` (Wrangler 4.45 or later; the build log you've seen use 4.163.0, so you're covered), Wrangler creates the namespace in your Cloudflare account automatically and links it to the Worker. Nothing to run by hand. This namespace is where the weekly stocktake data actually lives — there's no external service involved at all.

If you're ever on an older Wrangler and this doesn't happen automatically, run `npx wrangler kv namespace create CHEM_KV` yourself and paste the printed `id` into the `kv_namespaces` block in `wrangler.jsonc`.

### 2. Push this repo to GitHub

Create a new GitHub repo and push these files to it (a private repo is recommended, though nothing sensitive lives in the code itself since secrets are set separately in Cloudflare).

### 3. Connect it to Cloudflare via Workers Builds (Git integration) — or just deploy locally

Either connect the repo via **Workers & Pages → Create → Workers Builds**, or skip Git entirely and run `npx wrangler deploy` from your own machine once the KV namespace ID is in `wrangler.jsonc`. Local `wrangler deploy` has been the more reliable path for this project — see "If something's not working" below.

### 4. Set the two secrets

In the Worker's **Settings → Variables and Secrets**, add these as type **Secret** (not Text):

| Name | Value |
|---|---|
| `SITE_PASSWORD` | The shared password your team will type in to get past the login screen |
| `SESSION_SECRET` | A long random string (e.g. generate one with `openssl rand -base64 32`) — used to sign session cookies. Don't reuse this across the sibling sites, and never leave it empty. |

The KV namespace itself is a **binding**, configured in `wrangler.jsonc` (step 1), not a secret — there's nothing to add for it here.

After saving secrets, redeploy (`npx wrangler deploy`) and the site should come up behind the login screen.

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

- Check the Worker's logs in the Cloudflare dashboard (Observability is enabled in `wrangler.jsonc`). An exception on `/api/sync` with no useful detail usually means the `CHEM_KV` binding didn't get created/linked — confirm a KV namespace actually exists for this Worker under **Workers & Pages → chemicaltracker → Settings → Bindings**, and that the last deploy ran `npx wrangler deploy` (dashboard-triggered redeploys have previously broken binding setup for this project).
- The site still falls back to saving locally on that device if `/api/sync` is unreachable, so nobody loses their in-progress count even if sync is temporarily down.

## Updating the chemical list

The chemical master list (name, supplier, min stock, storage) lives near the top of the `<script>` block in `public/index.html`, in the `CHEMICALS` array. Adding a new chemical, retiring one (set `active: false` like `Cellarwash`), or changing a min stock level is a matter of editing that array and redeploying — there's no separate database for it, since the list changes rarely compared to the weekly counts.

Most chemicals are tracked as a plain count against a min stock level, and the suggested order tops back up to that level. **Synergex 1000L** is different — it's tracked as a percentage remaining in the IBC currently in use, and when it's opened there's no "topping up," just a full replacement. That's handled with three extra fields on its entry:

- `min: 30` — the reorder threshold is 30%, not a unit count
- `inputStep: 1` — the stocktake field steps by whole percent rather than 0.01
- `orderUnit: "x 1000L IBC"` and `fixedOrderQty: 1` — when below the threshold, the suggested order is a flat 1 new IBC, not "min minus on-hand"

The same pattern (`fixedOrderQty` + `orderUnit`) can be reused for any other chemical that gets swapped rather than topped up.

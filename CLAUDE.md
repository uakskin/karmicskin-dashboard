# Karmic Skin Dashboard

This repo (`uakskin/karmicskin-dashboard`) is the **single source of truth** for the
Karmic Skin dashboards. GitHub Pages serves it live at
https://uakskin.github.io/karmicskin-dashboard/.

## The one rule

Edit the dashboards **here, in this repo — nowhere else.** There is no separate
"source" copy. This repo is both the source and the live site.

Do **not** edit dashboard HTML in the `business-data-hub/` folder of the
`karmic-skin-claude` monorepo. Those copies are retired; any still there are stale.

## Workflow

1. `git pull` first — always start from the latest.
2. Edit the HTML files directly.
3. `git add` / `git commit` / `git push` — GitHub Pages redeploys automatically.

No build step. Each page is a self-contained single HTML file (CSS in one `<style>`,
JS in one `<script>`).

## Pages

- `dashboard.html` + `index.html` — Shopify dashboard (`index.html` is a copy of `dashboard.html`)
- `amazon.html`, `meta.html`, `email.html`, `pnl.html` — channel dashboards (`pnl.html` loads its figures from
  Supabase `pnl_monthly` at runtime; see "P&L data" below). `meta.html` (rebuilt 20 Oct 2026 on the design system)
  is the Meta overview: every figure is shown against the prior 90 days (the 90 days before the selected window);
  it reads `raw_meta_ads_daily`, `rpt_daily_overview`, `v_meta_ads_campaign_daily`, `raw_meta_ads_ad_daily`
  (window only, for ad counts and spend concentration), `meta_campaigns` (daily budgets) and `v_meta_ad_lifetime`.
  Ad-level tables live in `creative.html`, not here.
- `creative.html` — **Creative Analytics** (Motion-style creative reporting for the Meta account). Login gate is the
  `creative_access` table (roles `admin` / `strategist`), NOT `allowed_emails`: strategists see this page only, with no
  KS sidebar, and RLS limits them to the Meta tables. Data: `meta_ad_creatives`, `raw_meta_ads_ad_daily`,
  `v_meta_ad_lifetime`, breakdown snapshots; thumbnails in the private `creative-media` bucket (signed URLs);
  video via the `creative-media` Edge Function. Saved reports / tags / settings live in `creative_reports`,
  `creative_tags`, `creative_settings`. Sync: `sync-meta-creatives` hourly, `sync-meta-ads` every 30 min,
  breakdowns nightly. See memory `project_creative_analytics_app`.
  **Deployment rule (Uday, 19 Oct 2026): every change to `creative.html` must reach every surface strategists
  use, in the same session, without being reminded. Today that is this one file at this one URL (role decides
  the view). If a separate strategist portal is ever added (other repo, domain or host), push the identical
  file to BOTH, verify BOTH live, and record the second location here and in memory
  `feedback_creative_analytics_deploy_everywhere`. Never leave them out of sync.**
- `launch.html` — New Product Launch Gantt (half-month grid, editable, state in Supabase `launch_plan_state` row 1)
- `inventory.html`, `ugc.html`, `instagram-scheduler.html` — operations tools (Inventory sits under Analytics in the sidebar; `worklist.html` was removed 2026-10-10)
- `design-system.html` — brand design system (working draft)

## Shared sidebar nav

Every page carries an identical left-sidebar component: `<aside id="ksb">` with its
own `<style>` and `<script>`. Two sections — **Analytics** and **Operations**. The
active page is detected automatically from the filename (`data-page`).

Adding a page or changing the nav: apply the **same** sidebar block to **every**
page — it must stay identical across all of them.

## UK time (every page)

Every date and time a page shows or uses is **UK time** (Europe/London: GMT in winter, BST in
summer), whatever time zone the viewer's computer is in (Uday often views from India). Since
30 Oct 2026 each page that needs it carries a small `UK TIME` helper block:

- `ukNow()` — the current UK wall-clock time as a Date whose local getters read UK time. Use it
  instead of `new Date()` whenever you derive "today", "yesterday", month-to-date, the current
  hour, etc. Never use `new Date()` for that, and never `toISOString()` to get a calendar date
  (it is the UTC date).
- `fmtUkDateTime(ts, opts)` — show a stored timestamp in UK time with its zone ("14:07 GMT" /
  "BST"). For a timestamp's UK calendar date use `ukDateOf(ts)` (email, ugc) or `ukDay(s)`
  (creative). `YYYY-MM-DD` strings from the warehouse are already UK days: parse them as
  `new Date(s + 'T00:00:00')`.
- `instagram-scheduler.html`: datetime boxes are typed in UK time; convert with
  `ukInputToIso()` / `toLocalInput()`, never `new Date(boxValue)`.

The warehouse follows the same rule: daily tables and the order RPCs bucket by
`(created_at AT TIME ZONE 'Europe/London')::date`, never `::date` on a UTC timestamp or `CURRENT_DATE`.

## Amazon page data rules (`amazon.html`)

- Ad spend is all three Amazon ad types (`raw_amazon_ads_daily.ad_product` = SP / SB / SD),
  synced by `sync-amazon-ads` nightly (last 3 days) and hourly (today so far).
- Sales land every 30 min, ad spend hourly, traffic (sessions, page views) 1–2 days late. ROAS
  only counts revenue on days whose ad spend has loaded (`roasRevenue`), and conversion only
  counts units on days whose sessions have loaded (`cvrUnits`), so a fresh day can't inflate them.
- The header shows when the data last updated (UK time) and, if ad spend lags sales, how far it
  reaches; hover it for each feed's coverage.

## Security

Dashboards use the Supabase **anon key** only (public, RLS-protected). This repo is
public — never commit service-role keys or Shopify / Meta / Klaviyo / Amazon tokens.

### Admin gate (every page except creative.html)

Every page other than `creative.html` carries an identical `<script>` block in `<head>`, marked
`KS ADMIN GATE` (added 19 Oct 2026). Before anything paints it reads the magic-link session
`index.html` stores under localStorage `ks-dashboard-auth`, checks the signed-in email against
`allowed_emails` (the same REST self-read `index.html` does), and only then reveals the page.
No session, an expired session, or an email not on the list -> `index.html` (the login); a
`creative_access` strategist -> `creative.html`; a magic link that lands on a gated page is
forwarded to `index.html`, which adopts it. The gate also refreshes the token a minute before
it expires and announces the new one with a `ks-session` window event, which every page's
`USER_JWT` listens for. Keep the block **byte-identical across pages** (same rule as the
sidebar). `creative.html` keeps its own `creative_access` gate and is not part of this rule.

The gate is the front door, not the only lock: every table behind these pages is RLS-limited
to `public.is_allowed_user()` (allowed_emails) for the `authenticated` role, with no anon
policies. That includes `launch_plan_state`, `diary_state`, `ugc_creators`,
`mod_shopify_inventory`, `pnl_monthly`, `mod_orders`, `raw_shopify_refunds` and the daily spend
tables. `launch.html`, `diary.html` and `ugc.html` therefore send the admin's token, not the
anon key. Edge Functions use the service role and are unaffected.

### P&L data (`pnl.html`)

The P&L figures are **not in this repo** any more. The repo is public, so anything in the HTML
can be read with curl whether or not the viewer is logged in; a JavaScript gate hides nothing
from that. Since 19 Oct 2026 the figures live in Supabase table `pnl_monthly` (admin-only RLS):
one row per fiscal year and month, `fy` = `fy2025` / `fy2026`, `month_idx` 0 = June .. 11 = May,
`label` e.g. `May 26`, and total / Shopify / Amazon columns that mirror the old arrays:
`revenue`, `shop_rev`, `amz_rev`, `prod_cost`, `shop_prod_cost`, `amz_prod_cost`, `fulfilment`,
`shop_fulf`, `amz_fulf`, `ad_spend`, `shop_ads`, `amz_ads`, `fixed_subs`, `shop_fixed`,
`amz_fixed`, `other_exp`, `other_inc`, `ebitda`, `shop_ebitda`, `amz_ebitda`. `pnl.html` loads
the table at runtime with the signed-in admin's token and derives `activeMonths` from the row
count. To add or restate a month, upsert its row (SQL through the Supabase MCP or the service
role); never put figures back into the HTML. Commits before 19 Oct 2026 still carry the old
figures in `pnl.html`'s history.

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
- `inventory.html`, `ugc.html`, `instagram-scheduler.html` — operations tools (Inventory sits under Analytics in the sidebar; `worklist.html` was removed 2026-10-10).
  The packaging count (jars / bottles / cartons per SKU) is NOT in `inventory.html`: it lives in the
  admin-only table `inventory_packaging_stock` (one row per SKU, `received` = count date, `sort_order` =
  display order), moved out of the page source on 1 Nov 2026 so it is not public. Update the table
  when a new count sheet arrives; the page needs no edit.
- `design-system.html` — brand design system (working draft)
- `mobile.html` — **Topline**, Uday's phone view (added 1 Nov 2026), installed on his iPhone home screen as
  its own app (`mobile.webmanifest`, `mobile-icon-180/192/512.png`, scope = this folder, start = mobile.html).
  Six tiles: Shopify revenue, blended ROAS (vs 2.20, amber mark when below), repeat orders %, orders, AOV,
  Amazon sales (with "All Sales = Shopify + Amazon") and Amazon ROAS as its own tile (phone only), worked out
  exactly as amazon.html does: daily SP+SB+SD spend, monthly spend spread over sales days before 22 Dec 2025,
  sales counted only on days whose spend has landed); Today by default, WTD / MTD pills, and a Range sheet (yesterday, last month, YTD, current FY
  from 1 June, last 12 months, last year, plus one calendar where you tap the first and last day, then Apply).
  Uday's rules (3 Nov 2026): **no trend lines, and everything fits one screen with no scrolling** (the tile
  rows share the screen height; checked at 393x700 Safari and 402x874 home-screen); the method notes sit
  behind the (i) button. It reads `rpt_daily_overview` and `raw_amazon_sales_daily`
  (two light REST reads per range) with the same sums as `index.html` / `amazon.html`, so the figures
  match them; QA'd 1 Nov 2026 against both pages' own code on a fixture for eight ranges.
  **Shopify figures come from Shopify itself (4 Nov 2026):** revenue (Shopify "Total sales"), orders and
  returning-customer orders are read from Shopify Analytics (ShopifyQL `FROM sales SHOW orders, total_sales
  GROUP BY new_or_returning_customer`) through the admin-only Edge Function `shopify-sales` (checks the
  caller against allowed_emails; Shopify token in its secrets; CORS only for uakskin.github.io), so they match
  Shopify's reports exactly; the page re-asks every 15 s while a range includes today. Why: Shopify books order
  edits, returns and cancellations on the day they happen, while rebuild-reporting applies them to the order's
  own day, so the warehouse (and index.html) can be a few pounds and a few returning orders off Shopify.
  Ad spend still comes from `rpt_daily_overview`; Amazon unchanged. If `shopify-sales` fails, the page falls
  back to the warehouse path below and says so.
  **Fallback live orders:** for today it swaps in Shopify revenue, orders and repeat orders from the SQL function
  `get_shopify_live_day(p_day)` (SECURITY INVOKER, authenticated only), which computes them straight from
  `raw_shopify_orders` / `raw_shopify_refunds` with rebuild-reporting's exact rules (verified equal to the
  rebuilt rows for 8 days, ~20 ms). The Shopify webhook lands orders in seconds, and the page re-checks every
  15 s while open, so a new order shows within ~20-30 s; ad spend for today stays as last rebuilt (10 min).
  If you change how rebuild-reporting counts revenue or new/returning orders, change that function too. It carries the KS ADMIN
  GATE but deliberately **no sidebar** (phone app; exception to the shared-sidebar rule, like creative.html).
  Sign-in from the home-screen app: see "Phone sign-in" under Security.

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

## Shopify figures in the warehouse (since 4 Nov 2026)

`rpt_daily_overview`'s Shopify columns (total_sales, net_revenue, total_orders, orders_from_new/returning,
new/returning_customer_orders and _revenue, net_product_revenue, aov, total_discounts, total_refunds and the
percentages built on them) come from **Shopify's own daily figures**, so every page that reads the overview
(index, meta, email, creative, mobile, the weekly/monthly views and the day-of-week / LTV RPCs) matches
Shopify Analytics. `sync-shopify-sales` (pg_cron `sync-shopify-sales-10m` at :03/:13/..., last 3 days; nightly
`sync-shopify-sales-nightly` last 35 days) writes ShopifyQL `FROM sales ... GROUP BY day[, new_or_returning_customer]`
into `raw_shopify_sales_daily` (admin-only RLS, one row per UK day, zeros for quiet days); `rebuild-reporting`
(v44) uses those rows for its 7-day window and keeps the old order-based count (still in `mod_daily_sales`) as
the fallback for any day without Shopify figures, and for today when the Shopify row is over 30 minutes old.
History was switched in one UPDATE; the previous values are in `bak_rpt_daily_overview_20261104`.
Why: Shopify books order edits, returns and cancellations on the day they happen (e.g. Oct 2026 was
£73,688.74 / 1,363 orders order-based vs Shopify £73,756.93 / 1,364). Known Shopify quirk: an order with sales
activity on two days counts as an order on both days, so adding up days can be 1 order above Shopify's own
total for a long range (once in 2025: 5 Oct 2025). Percent columns are numeric(5,2): the rebuild stores 1000%+
as blank so one odd day can't fail the insert after the window was deleted.

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

**A Supabase hiccup never signs anyone out** (5 Nov 2026). A timeout, 5xx, 429 or dropped
connection (including an ISP blocking supabase.co) means Supabase didn't answer, not that the
session is bad: the gate, `index.html` and `creative.html` keep the stored session and retry
(gate: 15s, 30s, 60s, then every 2 min; index/creative: 15s, 30s, 1m, 2m, then every 5m, with a
"you're still signed in" note on the login card while it retries). Only a refresh token Supabase
rejects (400/401/403) clears the session and shows the login. The same goes for the
allowed_emails / creative_access checks: a 5xx there is retried, never read as "not on the list".
Why: on 5 Nov 2026 a 10-second Supabase Auth timeout made the old code delete a valid session;
the retries then hit the 2-emails-an-hour limit and Uday was locked out for an hour. Any new
page or auth change must keep this rule.

The gate is the front door, not the only lock: every table behind these pages is RLS-limited
to `public.is_allowed_user()` (allowed_emails) for the `authenticated` role, with no anon
policies. That includes `launch_plan_state`, `diary_state`, `ugc_creators`,
`mod_shopify_inventory`, `inventory_packaging_stock`, `pnl_monthly`, `mod_orders`, `raw_shopify_refunds` and the daily spend
tables. `launch.html`, `diary.html` and `ugc.html` therefore send the admin's token, not the
anon key. Edge Functions use the service role and are unaffected.

**Write every policy check as `(select public.is_allowed_user())`, never bare `is_allowed_user()`**
(same for `is_creative_user()`, `is_creative_admin()` and `auth.email()`). Bare, Postgres runs the check
once per row; wrapped, once per query. All 62 policies were rewritten this way on 5 Nov 2026 (migration
`rls_helper_checks_once_per_query`) after the bare form made the cohort/LTV/product reports 10x slower
(3-4 s each) and helped push the free-tier database into a disk stall. Who can see what is unchanged.

### Sign-in emails (custom SMTP since 5 Nov 2026)

Supabase sends the sign-in emails through **Gmail's mail server**, not its own built-in mailer (which allows only
2 emails an hour for the whole project and caused the 5 Nov lockout). Settings live in Supabase > Authentication >
Emails > SMTP Settings: host `smtp.gmail.com`, port 465, username `uday@karmicskin.com`, sender
"Karmic Skin <uday@karmicskin.com>". The password is a Google **app password** on Uday's account named
"Supabase sign-in emails"; it is stored only in Supabase, never in any file. If that app password is ever
revoked, or the uday@ account's password changes (Google then revokes app passwords), sign-in emails stop:
create a new app password and paste it into the same screen. SPF (`include:_spf.google.com`) and Google DKIM
already cover uday@, so these emails pass DMARC. Because Gmail sends them as uday@, every sign-in email
(anyone's, Aayushi's and Ananya's too) also appears in uday@'s Sent folder. That is expected.

The project-wide limit is now **30 sign-in emails an hour** (Authentication > Rate Limits), plus a 60-second gap
between emails to the same person. Past either, `/otp` answers 429 and the login explains it in plain words.
Saving SMTP or rate-limit settings restarts Supabase Auth, which took several minutes on 5 Nov (free plan,
Disk IO budget nearly used up): sign-ins fail during that window, existing sessions ride it out.

### Phone sign-in (`mobile.html` home-screen app)

An iPhone home-screen web app has its own storage, separate from Safari, and a magic link opens in Safari,
so a link can never sign the app in. Since 1 Nov 2026 `index.html`'s login therefore also takes the **code**
from the same email (`POST /auth/v1/verify`, `type: 'email'`). The code comes from `{{ .Token }}` in the
Supabase **Magic link or OTP** email template (Authentication > Emails), added 1 Nov 2026 under the link; it is
**8 digits** (Email OTP length setting) and lasts an hour. Don't remove it from the template, or the phone app
can't sign in. The **"I already have a code"** button under "Send sign-in email" goes straight to the code box
without sending another email. The email the code went to is
kept in localStorage `ks-pending-signin` for an hour so a reloaded app reopens on the code box.

Getting back to the phone view: before the gate runs, `mobile.html` leaves a note in localStorage
`ks-return-to` (`{page, at}`) when its session is not live, and always when running standalone. `index.html`
hides itself while a fresh note (under 30 minutes) is pending, and after a successful admin sign-in or token
refresh it reads and clears the note and goes back to that page (only pages in its `RETURN_PAGES` list)
before loading the dashboard. Without a note, `index.html` behaves exactly as before.

iOS saves whatever page is open when you tap Add to Home Screen, and a signed-out visit to mobile.html is
sitting on `index.html` (the login) by then. So `index.html`, when opened as an iPhone home-screen app
(`navigator.standalone === true` and a touch screen, which never happens in a desktop browser or a normal
Safari tab), stays hidden and always hands over to `mobile.html` after sign-in (added 1 Nov 2026, after
Uday's first icon opened the full dashboard).

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

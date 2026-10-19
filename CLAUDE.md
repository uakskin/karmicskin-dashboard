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
- `amazon.html`, `meta.html`, `email.html`, `pnl.html` — channel dashboards
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

## Security

Dashboards use the Supabase **anon key** only (public, RLS-protected). This repo is
public — never commit service-role keys or Shopify / Meta / Klaviyo / Amazon tokens.

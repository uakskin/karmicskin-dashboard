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
- `inventory.html`, `worklist.html`, `ugc.html`, `instagram-scheduler.html` — operations tools
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

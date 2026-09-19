# CLAUDE.md — deyderae-site

This repo is the apex site for `deyderae.dev`. Before making changes, read **`DESIGN.md`** in this repo — it's the design document for the whole `deyderae.dev` domain (all subdomains, not just this repo), covering architecture, security, and workflow standards. This file only holds conventions specific to working in *this* repo.

## What this repo is
Static HTML/CSS/JS landing page for `deyderae.dev` — no build step, no framework, deployed to Cloudflare Workers (static assets) via Wrangler. It links out to subdomain projects (currently `eclipse.deyderae.dev`).

## Working in this repo
- **Only `public/` is published.** `wrangler.jsonc` sets `assets.directory` to `./public`, so the site files (`index.html`, `style.css`, `script.js`, and any future `_headers`/images) live there and nothing else does. Wrangler does not skip `.git` or `node_modules` on its own, so never point `assets.directory` back at the repo root and never put non-site files (docs, config, tests) in `public/`.
- No package manager, no build step, no bundler. Don't introduce one for a trivial change — see DESIGN.md §3.2 for when it's actually warranted (this site stays plain HTML/CSS/JS unless it grows real interactive views).
- The theme system (Catppuccin, 4 palettes, `data-theme` attribute + `localStorage`) in `public/style.css`/`public/script.js` is currently duplicated in `eclipse-site`. Per DESIGN.md §3.4, that duplication is a known issue to fix, not a pattern to copy again for a new subdomain — check whether a shared source has been extracted before hand-copying these files elsewhere.
- Avoid inline `<script>`/event handlers (see DESIGN.md §4, §6.1) — it blocks a strict CSP. Put logic in `public/script.js`.
- `wrangler dev` for local preview; `wrangler deploy` for production (until CI/CD from DESIGN.md §6.3 exists — check whether that's landed before assuming manual deploy is still the process).
- No tests or CI exist yet. If you add either, follow DESIGN.md §6.3–§6.4 rather than inventing a one-off approach.

## Don't
- Don't commit secrets or Cloudflare tokens.
- Don't add a framework or router "for consistency" with other subdomains — this site doesn't need one; see DESIGN.md §3.2.
- Don't hand-edit deployed content directly in Cloudflare — changes go through git.

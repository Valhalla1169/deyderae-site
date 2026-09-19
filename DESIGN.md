# deyderae.dev — Design Document

**Status:** Living document. Update it whenever an architectural decision changes; don't let it drift from reality.
**Owner:** Caleb (Valhalla1169)
**Last reviewed:** 2026-09-19

## 1. Purpose and scope

This document defines how the `deyderae.dev` personal domain and its subdomains are designed, built, secured, and maintained. It exists because the project grew past "one HTML file" without anywhere to write down the decisions, so the same questions (how do we add a subdomain, where do secrets go, what's the deploy process) would otherwise get re-answered inconsistently each time.

It covers every property under `deyderae.dev` — the apex site and every subdomain — present and future. Repo-specific detail that doesn't matter domain-wide belongs in that repo's own `README.md`; this document holds the standards all of them share.

## 2. Current state (as of this audit)

| Property | Repo | Purpose | Status |
|---|---|---|---|
| `deyderae.dev` | `deyderae-site` (github.com/Valhalla1169/deyderae-site) | Landing page / hub linking to subdomain projects | Live |
| `eclipse.deyderae.dev` | `eclipse-site` (github.com/Valhalla1169/eclipse-site) | Live character-sheet hosting for a custom TTRPG: players use their own sheet, a DM can view any player's sheet | Placeholder ("coming soon"); backend planned on **Supabase** |

Both are currently:
- Static HTML/CSS/JS, no build step, no framework. (`deyderae-site` has a `package.json` only to pin Wrangler as a dev dependency; there are no runtime dependencies. See §5.4.)
- Deployed to **Cloudflare Workers** (static assets mode, via `wrangler.jsonc`), one Worker per repo. Only each repo's `public/` directory is published (§3.5).
- Styled with a hand-rolled **Catppuccin** 4-theme system (Latte/Frappé/Macchiato/Mocha) switched client-side via `data-theme` + `localStorage`, with the palette CSS and theme-switcher logic copy-pasted between the repos (§3.4). The apex site's `style.css` and `script.js` have since gained site-specific layout and footer-year code, so the files are no longer byte-identical.
- Pushed directly to `main` and deployed manually with `wrangler deploy` (`npm run deploy`), with no CI, no PR review step, no tests, no lint, and no security headers configured.

None of that is wrong for where the project is today — a static personal site doesn't need a build pipeline. It's called out here because several of these gaps become real risks the moment a subdomain (starting with Eclipse) grows into an actual application with user data. This document sets the bar to grow into, not a demand to rebuild what already works.

### 2.1 Standards check: apex site (2026-09-19)

Checked directly against the live site, the repo, and DNS. It is a dated snapshot, so re-verify rather than trusting it once anything changes.

| Standard | Status |
|---|---|
| §3.5 Only `public/` is published | Pass. Verified by `wrangler deploy --dry-run` (4 files) and live 404s for `/DESIGN.md`, `/wrangler.jsonc`, `/.git/config` |
| §4 Landmarks, heading order, focus states, `aria-pressed`/`aria-label`, reduced motion | Pass |
| §4 Contrast in all four themes (AA) | Pass. Computed from the palette values: lowest small-text ratio 4.7:1 (Latte footer text), large text at least 4.1:1 |
| §4 Legible without JavaScript | Pass (the theme buttons are inert without JS) |
| §4 No inline scripts or handlers | Pass. The page was also tested under a strict CSP with zero violations (§5.1) |
| §4 SEO: `<title>`, meta description | Pass |
| §4 SEO: Open Graph/Twitter tags, canonical URL, `sitemap.xml` | Missing |
| §4 `robots.txt` | Served, but it is Cloudflare's managed content-signals file, not a file in the repo |
| §5.1 Security headers (CSP, HSTS, nosniff, Referrer-Policy, Permissions-Policy, frame-ancestors) | Missing on the live site. A CSP is now unblocked |
| §5.2 DNSSEC | Pass (DS and DNSKEY published, resolver-validated) |
| §5.2 CAA records | Missing |
| §5.2 Registrar lock | Not verifiable from outside. Confirm at the registrar |
| §5.3 No secrets committed, `.gitignore` present | Pass (nothing sensitive tracked or in history) |
| §5.4 Dependency hygiene | `npm audit` reports 0 vulnerabilities. Dependabot and a CI audit step are not set up |
| §6.1 PRs, protected `main`, Conventional Commits | Not in use (direct pushes; commit messages are not Conventional Commits style) |
| §6.2 README and LICENSE | README is a title only, and there is no LICENSE. `package.json` says `"license": "ISC"`, which is npm's default rather than a deliberate choice (§8.1) |
| §6.3-§6.4 CI/CD, previews, link check, HTML validation, Lighthouse, axe | None yet. Deploys are manual |

## 3. Architecture

### 3.1 One SPA per (sub)domain, independently deployed

Each domain — the apex and every subdomain — is its own single-page application, in its own repository, deployed as its own Cloudflare Workers (or Pages) project. This is already the shape in place (`deyderae-site`, `eclipse-site`) and it stays the model going forward:

- **Independent deploys.** Shipping Eclipse never risks the apex site or any other subdomain, and vice versa. Each Worker has its own rollback history.
- **Independent blast radius.** A vulnerability or outage in one subdomain's app doesn't propagate to another's — they don't share a runtime, a database, or credentials unless explicitly wired together (see §6.6).
- **Independent tech choices, within shared standards.** The apex site can stay plain HTML/CSS/JS forever; Eclipse can adopt a framework once it needs real interactivity and state. Both still owe conformance to this document (accessibility, security headers, CI, etc.) — the standards are shared even when the implementation isn't.

A new subdomain gets its own repo and Worker project, not a route bolted onto an existing one. Routing within a single subdomain's SPA is fine; routing *between* subdomains happens at the DNS/link level, not by merging codebases.

### 3.2 "Single page application" — what that means here

For a purely presentational site (the apex landing page today), "SPA" just means one HTML document, no full page reloads for state that doesn't need them (theme switching, etc.) — plain HTML/CSS/JS is the right tool and no framework is required.

Once a subdomain has multiple *views* a user navigates between (Eclipse will: a character list, a character sheet, an editor), it becomes a true client-routed SPA, which raises requirements the current sites don't have to think about yet:

- **A router.** Given the scale (a handful of views, not a large app), prefer a small, dependency-light approach: either a minimal hand-rolled hash or History API router, or a lightweight framework (Preact, Svelte, Solid — chosen for bundle size over React's, since these are personal, performance-conscious sites). Pick per-subdomain based on that subdomain's actual complexity; don't standardize on a heavy framework domain-wide "just in case."
- **Server-side fallback for deep links.** Cloudflare Workers static-assets deployments must be configured so that unknown paths fall back to `index.html` (a `not_found_handling: "single-page-application"` setting in `wrangler.jsonc`, or equivalent), otherwise refreshing on `/characters/123` 404s instead of letting the client router take over.
- **Real, shareable URLs.** Every view a user might want to bookmark or share gets a real path via the History API — not everything hidden behind JS state with no URL to show for it.
- **A loading/empty/error state for every async view.** Once there's a data layer (§3.3), every screen that fetches something needs to visibly handle "still loading," "nothing here yet," and "that failed" — not just the happy path.
- **Live/realtime views, where the feature actually calls for it.** Eclipse's DM-visibility requirement (§3.3.1) means at least one screen — a DM watching a player's sheet — benefits from updating without a manual refresh. Reach for a realtime subscription only on the views that need it (a DM's live view of a sheet); a player's own sheet editor can be a normal fetch/save loop unless there's a real multi-device-at-once case.

### 3.3 Data and API layer (for when a subdomain needs one)

The general default for a subdomain needing persistence: Cloudflare **D1** (SQLite) for relational data, **KV** for simple key-value/cache use, **R2** if file/image storage is ever needed, kept in that subdomain's own Worker (§6.6). That default doesn't apply to Eclipse — see §3.3.1 — because Eclipse already has a specific reason to reach for something else.

- **Schema migrations live in the repo**, versioned and reviewed like code — never hand-edited against production, regardless of which datastore a subdomain uses.
- **API surface:** if application logic needs a backend beyond what the datastore itself provides, keep it in that subdomain's own Worker rather than introducing a separate service, unless and until multiple subdomains need to share the same backend logic or data (e.g., a shared login across subdomains). If that happens, factor it into a dedicated `api.deyderae.dev` Worker with its own repo, and have subdomains call it — don't let two subdomains reach into the same database directly.
- **Auth**, for any subdomain that isn't Eclipse: prefer an existing provider (Cloudflare Access for anything gated to just you; a proper auth provider for anything with real user accounts) over hand-rolled session/password handling. Passwords and sessions are exactly the kind of thing not to build from scratch on a personal project.

#### 3.3.1 Eclipse specifically: Supabase, and the DM/player access model

Eclipse's actual requirements are more specific than "store some character data": players need live access to *their own* sheet, and a DM needs to be able to view *any* player's sheet under their campaign. That's a real authorization model (two roles, row-level ownership, a cross-user read grant), not just CRUD behind a login — and it's also live/collaborative in a way the apex site never is. **Supabase** (Postgres + Auth + Realtime) is the planned fit for exactly that shape of problem, and is the intended backend for this subdomain specifically — it does not replace D1 as the domain-wide default in §3.3, it's a deliberate exception for this one subdomain's requirements.

What that implies concretely:

- **Row Level Security (RLS) is the actual access-control boundary, not application code.** Supabase's client libraries talk to Postgres directly from the browser using a public `anon` key, so RLS policies — not a Worker, not client-side checks — are what actually stops one player from reading another player's sheet or writing to a sheet they don't own. Design the policies before writing UI against them:
  - A `characters` (or `sheets`) table with an `owner_id` referencing the authenticated user.
  - A `campaigns` table with a `dm_id`, and a join table (`campaign_players`) linking players to the campaigns they're in.
  - RLS on `characters`: a row is readable by its `owner_id`, *and* readable by the `dm_id` of the campaign it belongs to; writable only by its `owner_id` (decide explicitly whether a DM can ever write to a player's sheet, e.g. for adjustments — if so, that's a separate, narrower policy, not a blanket DM-write grant).
  - Write this as SQL migrations committed to the repo (§3.3, "schema migrations live in the repo") — Supabase supports this directly (`supabase/migrations`), so there's no excuse to hand-edit the schema in the dashboard.
- **Auth:** Supabase Auth (email/password and/or magic link, or an OAuth provider) issues the identity that RLS policies key off of. This *is* Eclipse's auth provider — no separate auth system needed alongside it.
- **Realtime, scoped narrowly.** Use Supabase Realtime (Postgres change subscriptions) for the DM's live view of a player's sheet, per §3.2. Subscriptions still go through the same RLS policies — a realtime subscription can't read what a plain query couldn't.
- **Secrets:** the Supabase `anon` key is meant to be public (it's shipped to the browser) and is *not* a secret — RLS is what makes that safe. The Supabase **service role key**, which bypasses RLS entirely, is a real secret: it must never reach client-side code, and only belongs in a trusted server context (a Worker route, if Eclipse ever needs one, via `wrangler secret`) for the specific operations that genuinely need to bypass RLS (e.g., admin tooling) — not as a shortcut around writing the right policy.
- **This is an external, third-party dependency**, unlike the rest of this domain's Cloudflare-only stack so far. That's a fine trade for what Supabase buys here (Postgres + Auth + Realtime, integrated, without hand-building any of the three), but it's worth naming as a dependency: Eclipse's availability now depends on Supabase's, and its data lives outside Cloudflare, which matters for backup/export planning and for understanding the actual blast radius if that account is ever compromised.

### 3.4 Shared design system — stop duplicating it

The Catppuccin palette CSS and the theme-switcher logic in `script.js` are currently copy-pasted across both repos. That's the first thing to fix, independent of everything else in this document, because every future subdomain will otherwise copy-paste it again and they *will* drift out of sync (a palette tweak in one site silently not applied to another).

Recommended fix: extract the shared pieces (CSS custom-property theme definitions, the theme-switcher script, the base reset/typography, the favicon) into one versioned source of truth, and pull it into each repo rather than hand-copying. In order of effort:

1. **Cheapest now:** a small internal npm package (or even just a `git subtree`/submodule) published from a new `deyderae-design` repo, consumed by each site's build.
2. **No-build-step-friendly alternative:** since these sites currently have no bundler, a single shared static asset (e.g., `theme.css` + `theme.js`) hosted at a stable URL (e.g., served from the apex domain or a tiny dedicated Worker) and `<link>`/`<script src>`-ed from every subdomain. Simple, but couples every subdomain's page load to that asset's availability — acceptable for a personal site, worth knowing about.
3. **When a framework and build step exist:** promote it to a proper shared component library.

Either way: one canonical place for the four palettes and the switching logic, every site references it, nobody hand-edits a copy again.

To keep that extraction mechanical, `deyderae-site/public/style.css` is split at a `Site layout` comment: everything above it is the shared theme (the four palettes), everything below is apex-specific (header and footer bands, hero, cards). Likewise `script.js` keeps the theme logic separate from the footer-year snippet. Until the shared source exists, don't edit the palette block or the theme logic in one repo only. Header, footer, and card styles should move into the shared piece only if a second site actually adopts them. The logo/favicon mark (`public/logo.svg`) is the other shared candidate.

### 3.5 What gets published: the `public/` directory

Each repo serves a dedicated `public/` directory, not the repo root. `wrangler.jsonc` sets `assets.directory` to `./public`, and only site content (HTML, CSS, JS, images, and the `_headers`/`_redirects` files) lives there. Config, docs, `package.json`, and tooling stay at the repo root.

- **Why a directory and not an ignore file.** Wrangler's asset upload walks the whole assets directory and skips only `.assetsignore`, `_redirects`, and `_headers`. It does *not* skip `.git` or `node_modules`. With `"directory": "./"`, `wrangler deploy` fails as soon as dependencies are installed (`node_modules` contains a ~90 MiB `workerd` binary, over the 25 MiB asset limit), and without them it would publish `.git`, `DESIGN.md`, `CLAUDE.md`, and `wrangler.jsonc` as public URLs. A root `.assetsignore` is a denylist that has to be updated for every new non-site file, whereas a dedicated folder cannot leak anything that isn't inside it.
- **Verify on every new subdomain.** `wrangler deploy --dry-run` should report `Read N files from the assets directory ...\public`, where N is exactly the number of site files. After a deploy, `/DESIGN.md`, `/wrangler.jsonc`, and `/.git/config` on the live site should return 404.
- **Never** point `assets.directory` back at the repo root, and never put non-site files in `public/`.

## 4. Frontend standards

These apply to every subdomain, static or app-like:

- **Accessibility — target WCAG 2.1 AA.** Semantic landmarks (`header`, `main`, `nav`, `footer`), correct heading order, visible focus states, sufficient color contrast in *every* theme (verify Latte's light palette against WCAG contrast ratios specifically — light themes are the easiest to accidentally fail), `aria-pressed`/`aria-label` on interactive controls (already used correctly on the theme buttons — keep that habit), and respect for `prefers-reduced-motion` and `prefers-color-scheme` (already partially done via the light/dark theme default — keep it). In practice, on Latte the `--subtext0` and `--overlay0` to `--overlay2` tokens, raw `--blue` as link text, and raw `--peach`/`--green` as text all fall below 4.5:1 on their usual backgrounds. Use `--text`/`--subtext1` for small text, and mix accent hues toward `--text` (for example `color-mix(in srgb, var(--blue) 75%, var(--text))`) when a coloured link or badge is needed. Large text (24px, or 18.66px bold) only needs 3:1. Check every theme, not just Mocha.
- **Progressive baseline.** A page should render its core content and be legible without JavaScript; JS enhances (theme switching, interactivity) rather than being required to see anything at all, wherever that's practical.
- **Performance budget.** Static pages: no framework, minimal payload, no render-blocking third-party scripts. App-like pages: track bundle size deliberately once a framework is introduced; a personal project has no excuse for a bloated bundle. A tiny first-party script that must run before first paint (theme selection) may load synchronously from `<head>` so the stored theme applies without a flash of the wrong palette; everything else should defer. Lighthouse performance/accessibility/best-practices/SEO scores are a useful cheap check — see §6.4.
- **SEO basics** on every public page: a real `<title>` and `<meta description>` (already present), Open Graph / Twitter Card tags for link previews, a `sitemap.xml` and `robots.txt`, and a canonical URL.
- **No inline event handlers or inline `<script>` blocks going forward** (the apex page's copyright-year script now lives in `script.js`; confirm `eclipse-site` is clean too) — move logic into the external JS file. This isn't just style: it's what makes a strict Content-Security-Policy possible (§6.2).

## 5. Security

Security gets real emphasis here because a personal domain is still a public attack surface, and it's much cheaper to bake this in now than to retrofit it once Eclipse holds other people's data.

### 5.1 Transport and headers
- HTTPS is already enforced by Cloudflare — keep "Always Use HTTPS" / HSTS on at the zone level, and add `Strict-Transport-Security` at the application layer too (defense in depth).
- Add security headers to every deployed site (via a `_headers` file for Workers static assets, or Worker middleware): `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` (deny what isn't used — camera, mic, geolocation, etc.), and `X-Frame-Options: DENY` (or `frame-ancestors 'none'` in CSP). None of this exists today; it's a same-day fix once §4's "no inline scripts" rule is in place, since that's what unblocks a CSP without `unsafe-inline`.
- **Apex readiness (2026-09-19):** the apex page has no inline `<script>`, no `style=` attributes, and no third-party origins. Served with `Content-Security-Policy: default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`, it loaded, applied a theme, and logged zero violations, so a `public/_headers` file can be added now. A subdomain that adds an external origin (Supabase for Eclipse, a font host) should extend `connect-src`/`font-src`/etc. deliberately rather than loosening the policy to `unsafe-inline`.

### 5.2 Domain and DNS
- Registrar lock enabled; DNSSEC enabled on the zone.
- CAA records restricting which CAs may issue certs for `deyderae.dev` and its subdomains.
- Keep DNS management (Cloudflare) and code deploy credentials (GitHub/Wrangler) separately scoped — don't reuse one token for both.
- **Status (2026-09-19):** DNSSEC is enabled and validating (DS and DNSKEY records published). No CAA records exist yet. Registrar lock can't be checked from DNS, so confirm it at the registrar.

### 5.3 Secrets and credentials
- No secrets, API tokens, or credentials ever committed to a repo. Use `wrangler secret` for anything a Worker needs at runtime; use GitHub Actions encrypted secrets for CI/CD credentials (e.g., the Cloudflare API token used to deploy).
- Every repo keeps a `.gitignore` covering at least `node_modules/`, `.wrangler/`, `.env*` (allowing `*.example`), `.dev.vars*`, `dist/`, logs, and OS/editor files. Both repos have one now; keep it in place before any local env files or build output show up.
- Rotate the Cloudflare API token used for deploys if it's ever pasted anywhere outside a secrets manager (a chat, a script, a README).
- **Supabase (Eclipse only):** the `anon` key is public by design and fine to ship client-side — it is not the security boundary, RLS is (§3.3.1). The **service role key** bypasses RLS entirely and is a genuine secret: never in client code, never committed, never logged; if it's ever needed at all, it lives in `wrangler secret` behind a specific server-side route, not in the SPA bundle.

### 5.4 Dependency hygiene (once dependencies exist)
The moment a `package.json` shows up (a build step, a framework, a shared design package): enable Dependabot (or Renovate) for automatic update PRs, and run `npm audit` (or `osv-scanner`) in CI on every PR. Pin versions; don't float on `latest`. **Status:** `deyderae-site` now has a `package.json` (Wrangler as a devDependency), so this section applies to it. `npm audit` reports 0 vulnerabilities (2026-09-19), but Dependabot and a CI audit step aren't set up yet, and `package.json` declares `wrangler` as `^4.135.0`. The lockfile pins the resolved version; tighten the range if you want the manifest itself pinned.

### 5.5 Application-layer security (once there's a data layer)
For Eclipse and any future app-like subdomain:
- Validate and sanitize all input server-side (for Eclipse, that means at the database via constraints/RLS/`CHECK` clauses, not just client-side form validation) — never trust the client, even for a solo-maintained hobby project.
- Rate-limit any endpoint that writes data or handles auth (Cloudflare has built-in rate limiting rules available at the edge; Supabase also rate-limits auth endpoints itself).
- Principle of least privilege on database/API bindings — a Worker should only have access to the D1 database / KV namespace it actually needs, not a shared blanket credential.
- If accounts exist, follow standard practice for password/session handling (or better: delegate to an established auth provider, per §3.3) — hashed+salted credentials if self-hosting auth at all, short-lived sessions, no sensitive data in JWTs beyond an identifier.
- **For Eclipse specifically: treat every new table as a security decision, not just a schema change.** A table with no RLS policy (or a permissive default one) is a data leak the moment it ships — write and test the policy in the same change that adds the table, and verify it by testing as a second, non-owning user/role, not just as yourself.

### 5.6 Subdomain isolation
Because each subdomain is its own Worker with its own bindings, a compromise or bug in one doesn't automatically expose another's data — keep it that way. Don't introduce a shared database credential used directly by multiple subdomains' Workers (route through a dedicated API service instead, per §3.3, if sharing is ever needed). Eclipse's Supabase project is its own boundary too: nothing else on the domain should hold its service role key or read its tables directly.

## 6. Development workflow

None of this exists yet (everything currently pushes straight to `main`), so this is the target to move toward, not a description of today:

### 6.1 Branching and review
- Trunk-based development: short-lived feature branches, merged via pull request — even solo, a PR is a checkpoint to actually re-read the diff, and it's where CI gates the merge.
- `main` is always deployable. Protect it: require the CI checks in §6.3 to pass before merge, and disable direct pushes.
- Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, …) for commit messages — cheap, and it means a changelog can eventually be generated instead of written by hand.

### 6.2 Repository hygiene
- Every repo gets: a real `README.md` (what it is, how to run it locally, how it's deployed — today's READMEs are just a title), a `.gitignore`, a `LICENSE` (decide whether these are meant to be open-source; if so, pick one explicitly), and a `CLAUDE.md` pointing back to this document (see §9).
- A shared issue/PR template isn't necessary for a solo project but a lightweight PR checklist (did you test it, did you check the deployed preview, does it pass CI) is worth keeping in the PR template once CI exists.

### 6.3 CI/CD
Add a GitHub Actions workflow per repo that, on every PR:
1. Lints (HTML validation at minimum; ESLint/Prettier once there's enough JS to warrant it).
2. Runs any tests that exist (§6.4).
3. Deploys a **preview** (Cloudflare Workers/Pages preview URLs support this natively) so a change can be reviewed live before merging, not just read as a diff.

On merge to `main`: deploy to production automatically via the same workflow (Wrangler's GitHub Action). This turns "deploy" from a manual `wrangler deploy` run locally into a repeatable, auditable step — and removes the temptation to deploy an uncommitted local change.

### 6.4 Testing
Right-sized for what exists today, growing with the project:
- **Now (static sites):** automated link-checking and HTML validation cost almost nothing to add and catch real mistakes (broken links, invalid markup).
- **Lighthouse CI** on every deploy — performance, accessibility, best-practices, and SEO scores tracked over time, not just eyeballed occasionally.
- **Accessibility checks** (axe-core, via a CI step) once there's enough markup to be worth automating.
- **Once real application logic exists** (Eclipse's character-sheet logic, any data validation, any router): unit tests (Vitest pairs well with Cloudflare Workers) for that logic, and basic end-to-end tests (Playwright) for critical user flows (create a character, save it, reload and see it persisted).
- **Eclipse's RLS policies specifically need their own tests**, separate from UI end-to-end tests: a policy that "works" because the UI never asks for the wrong row is not verified. Test as multiple distinct users/roles (a player reading their own sheet, that same player attempting another player's sheet, the DM reading a sheet in their campaign, the DM attempting a sheet outside their campaign) and assert both the allowed and the denied cases. Supabase supports running these against a local/CI Postgres instance so this can be a real CI check, not a manual spot-check.

### 6.5 Local development
- `wrangler dev` for local iteration against each Worker, matching production behavior (including assets routing) closely.
- Document the local setup steps in each repo's `README.md` — "clone, install, run" should not require asking future-you how it worked.

### 6.6 Environments
Three tiers, once CI/CD is in place: **local** (`wrangler dev`), **preview** (automatic per-PR Cloudflare preview URL), **production** (`main`, auto-deployed on merge). Keep any environment-specific config (API base URLs, feature flags) out of source and driven by Wrangler environment config / secrets instead of hardcoded values that differ by branch.

## 7. Forward compatibility

- **New-subdomain checklist / template.** Once the shared design system (§3.4) and CI workflow (§6.3) exist, turn them into a template repo (GitHub's native "template repository" feature) so spinning up the next subdomain is "generate from template, update the name," not "copy `eclipse-site` and hope you stripped out everything Eclipse-specific."
- **Framework choice stays per-subdomain, standards don't.** Don't force every future subdomain onto the same framework just for consistency; do force every future subdomain through the same accessibility, security-header, and CI bar.
- **Decisions worth writing down get an ADR**, not just a Slack-message-to-yourself. A short `docs/adr/NNNN-title.md` per significant choice ("why Cloudflare Workers over Pages," "why D1 over an external Postgres," "why no frontend framework on the apex site") is cheap and pays off the next time the same question resurfaces and the reasoning has been forgotten.
- **This document itself.** Review it whenever a decision it describes actually changes (a new subdomain, a new data store, a framework adoption) — a design doc that isn't updated becomes actively misleading, which is worse than not having one.

## 8. Open questions

These are flagged rather than decided, because they're genuinely this project's calls to make, not something to default silently:

1. **Are these repos meant to be public/open-source?** Affects licensing, whether secrets-in-history matters retroactively, and whether contribution guidelines are worth writing.
2. **Shared design system approach** (§3.4) — internal package vs. shared static asset vs. wait until a framework exists.
3. **CI provider** — this document assumes GitHub Actions since the repos are on GitHub; confirm that's still the intent before wiring it up.
4. **Eclipse's Supabase Auth method** — email/password, magic link, and/or OAuth — and how a DM's campaign/players actually get linked together (an invite code/link a DM shares, a player-requests-to-join flow, or the DM adding players by email) — this decides the shape of the `campaigns`/`campaign_players` tables in §3.3.1, so worth settling before writing that schema.
5. **Whether a DM can ever edit a player's sheet** (adjustments, corrections) or is strictly view-only — changes whether §3.3.1's write policy is single-owner-only or needs a narrower DM-write exception.

## 9. Companion file: CLAUDE.md

Each repo also gets a short `CLAUDE.md` (or `AGENTS.md`) that points back to this document and states any repo-specific conventions, so that AI-assisted changes (via Claude Code or otherwise) default to following these standards instead of re-deriving them per session. See the `CLAUDE.md` added alongside this file.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A chatbot-first variant of the **ePluribus** state services demo. After license verification, the user lands on a dashboard whose entire content is the **ePluribus AI** chatbot — no Quick Services tile grid, no Recent Activity / For You panels. Every other service (Wallet, State Taxes, Health Insurance, Benefits, Business, Voting, Jobs, Report It, Tickets, Get Involved) is still reachable through the sidebar, but the visual emphasis is unambiguously on conversation.

Companion variant of `civic-usa-3`, which is the dashboard-first Diia layout. This `civic-usa-4` exists to compare a chat-driven primary surface against a tile-driven one.

There is no build step, no package manager, and no dependencies. Visual design follows Diia: pure-white tiles floating on cool light gray with soft drop shadows, pastel-tinted icons, per-state accents, bold black for primary action buttons. The license card and wallet doc cards keep their richer self-contained designs.

## Running the site

The site **must be served over HTTP** (not opened as `file://`) — `sessionStorage` and relative links are unreliable otherwise.

```bash
cd civic-usa-4
python3 -m http.server 8768 --bind 127.0.0.1
# then open http://127.0.0.1:8768
```

(civic-usa-3, the tile-dashboard variant, runs at port 8767 alongside this one.)

Any static file server works (`npx serve`, `caddy file-server`, etc.); pick whatever's already installed.


## Architecture

### Page identity & shared layout

Every HTML file declares its identity via `<body data-page="...">`. The valid values are: `landing`, `scan`, `dashboard`, `license`, `taxes`, `healthcare`, `welfare`, `business`.

Signed-in pages contain a placeholder `<div id="sidebar-mount"></div>` and load `js/app.js`. On `DOMContentLoaded`, app.js:

1. Reads `document.body.dataset.page`.
2. If the page is in the `signedInPages` allowlist, calls `requireAuth()` (redirects to `index.html` when no user is in `sessionStorage`).
3. Replaces `#sidebar-mount` with the sidebar HTML from `renderSidebar(activePage)`.

`landing` and `scan` use a top-nav layout instead — they have no `#sidebar-mount` element.

### "Auth" state

Two `sessionStorage` keys, set during the scan flow and cleared on sign-out:

- `civic_state` — 2-letter code (e.g. `"CA"`)
- `civic_user` — JSON blob produced by `buildMockCitizen(stateCode)` in `app.js`

The user object is the single source of truth for the citizen's name, license number, address, expiry, etc. on every signed-in page.

### Per-state behavior

`app.js` exposes a few state lookup tables that pages consult:

- `STATES` — full name map for all 50 states
- `POPULAR_STATES` — order shown first in the landing dropdown
- `STATE_CITIES` — drives generated address in `buildMockCitizen`
- `STATE_TAX` — `{ rate, hasIncomeTax }` per state (with a `default` key); `taxes.html` branches on `hasIncomeTax` to show a "no state income tax" card for TX/FL/WA/etc., and uses `rate` to compute mock refund/owed numbers
- `buildMockCitizen` also picks a state-appropriate license-number format (CA → `D` + 7 digits, NY → grouped 9 digits, etc.)

When adding state-specific behavior, extend these tables rather than branching inline in pages.

### Where mock data lives

- **Cross-page identity / state-dependent values:** `app.js` (citizen object, tax rates, currency/date formatters)
- **Page-specific demo content** (appointments, prescriptions, filings, license entries, activity feed): hardcoded in each page's HTML, with at most a small `<script>` at the bottom that pulls the citizen object via `getCitizen()` and substitutes a few fields. There is no central content store.

### Icon system

Two parallel approaches:

- **Static HTML uses literal inline SVG** for hero/feature/list icons.
- **JS-rendered HTML uses `icon(name, size)`** from app.js (sidebar nav and the user-card avatar use this).

When extending the sidebar or other JS-rendered chrome, add to the `icons` object in `app.js`. When dropping an icon into static HTML, just paste the SVG.

## Adding a new service page

1. Create `<name>.html`. Copy the shell from `dashboard.html` (top-level `<div class="app">` with `<div id="sidebar-mount"></div>` + `<main class="main-content">`).
2. Set `<body data-page="<name>">`.
3. Add the page name to the `signedInPages` array in `app.js` (the `DOMContentLoaded` handler).
4. Add a sidebar entry inside `renderSidebar()` in `app.js`: `<a href="<name>.html" data-page="<name>">${icon('...')}<span>Label</span></a>`.

The sidebar's `setActiveNav(page)` call uses the `data-page` attribute to highlight the active link, so all four `data-page` values must match.

## Design tokens

`css/styles.css` defines the design system as CSS custom properties at `:root` (colors, radii, shadows, the primary gradient). Component classes (`.btn`, `.card`, `.stat`, `.service-card`, `.list-item`, `.badge-*`, `.license-card`) are reused across pages. Prefer adjusting tokens or composing existing classes over adding new ones.

The `.license-card` block specifically renders a credit-card-aspect digital driver's license with a US flag chip, portrait placeholder, and engraved field labels — it's the visual showpiece and is intentionally self-contained.

## Conventions

- No frameworks, no transpilation. Browser-targeted ES2017+ vanilla JS.
- Inline `<script>` tags at the bottom of each page handle page-specific glue. The shared `app.js` only contains things used by 2+ pages.
- All copy assumes a demonstration context — keep the "Demo data" disclaimer banners and the simulated-language in tooltips/alerts.

## Git workflow — commit locally, do not push

**Never let working changes pile up locally.** Treat git as the durable record of progress so a crashed session, lost shell, or context reset never costs us work.

- After completing each logical unit of work (a new page, a fix, a refactor, a tweak the user signed off on), `git add` the relevant files and create a commit. Don't batch unrelated changes into one commit.
- **Do not `git push` automatically.** The user pushes to the GitHub remote on their own cadence. Only push if the user explicitly asks for it in that turn.
- Write clean, conventional commit messages: a short imperative subject (≤ 70 chars) describing the *why*, followed by a blank line and a body only when the why isn't obvious from the subject. Examples:
  - `Add welfare benefits page with eligibility table`
  - `Fix license card flag rendering on mobile`
  - `Switch tax page to per-state rate table`
- Stage files explicitly by name (`git add scan.html js/app.js`) rather than `git add -A` / `git add .` to avoid sweeping in stray files.
- Never use `--no-verify`, `--force`, or `--amend` on already-pushed commits without explicit user approval. New commits over old ones; never rewrite shared history.
- If a commit fails (hook, conflict), stop and surface it — don't paper over with destructive flags.

If `git status` shows the repo is not initialized or has no `origin` remote, ask the user how they'd like to set it up before assuming. Do not silently `git init` or create remotes.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A static marketing site for **Bethesda AI** (brand name "BethesdaAI"; legal name Kimura Consulting LLC), plus two small Cloudflare Workers that back a couple of live-AI chat demos. There is no build system, no package manager, no bundler, and no test suite — every page is a single self-contained `.html` file with inline `<style>` and `<script>`. Editing is done directly on the HTML/CSS/JS in place.

The site was originally "BaltimoreAI" and was rebranded to "BethesdaAI" (teal palette → charcoal/cream/burnt-orange palette). The rebrand was applied page-by-page over several sessions, so when making sitewide changes, grep across *all* HTML files rather than assuming one page's state reflects another's — pages have drifted out of sync before (e.g. `demos.html` and its sub-pages remained on the old teal look and old "BaltimoreAI" copy for a while after `index.html` and `demos/legal.html` had already moved to the new brand).

## Commands

There is no build/lint/test tooling in this repo. Common tasks:

- **Preview locally**: `python3 -m http.server 8000` from the repo root, then open `http://localhost:8000/index.html` (or any page). There's no dev server with hot reload.
- **Deploy a Worker**: from `worker/`, `wrangler deploy` (civic/demos worker) or `wrangler deploy --config wrangler-legal.toml` (legal demo worker). Set the Anthropic key first: `wrangler secret put ANTHROPIC_API_KEY [--config wrangler-legal.toml]`.
- **Verify a change renders correctly**: there's no visual regression tooling; the practical approach used in past sessions is a headless Playwright screenshot (`chromium` binary preinstalled at `/opt/pw-browsers/chromium` in the Claude Code web/cloud environment; use `executablePath` to point at it, and `NODE_PATH=$(npm root -g)` if the `playwright` npm package isn't in this project's own `node_modules`).

## Architecture

### No shared templates — every page duplicates its own chrome

Each top-level page (`index.html`, `demos.html`, `privacy.html`, `terms.html`) and each page under `demos/` (`legal.html`, `eval-lab.html`, `knowledge-assistant.html`, `pipeline-runner.html`, `document-processor.html`, plus the `demos/index.html` redirect stub) carries its **own copy** of the navbar, footer, CSS custom-property palette (`:root { --brand-teal: ...; --brand-accent: ...; --ink: ...; ... }`), and favicon `<link>`. There is no include/partial mechanism, no shared CSS file, no templating engine.

**Implication**: any sitewide change — brand palette, footer copy, nav structure, favicon — has to be applied to every HTML file individually. `privacy.html`/`terms.html` are a legacy, unrelated product ("SmartText Assistant") that happen to live in this repo/domain but are `noindex,nofollow` and intentionally not styled to match the current brand; don't assume they should be kept in lockstep with the rest of the site's copy, only its favicon.

Because of this duplication, when doing a sitewide text/style change it's usually fastest to write a small Python script that does exact-string replacements per file (with an assertion on expected occurrence count) rather than editing each file by hand — that's the pattern used for the teal→charcoal rebrand and the nav-dropdown rollout.

### Brand palette variable names are historical, not literal

CSS custom properties like `--brand-teal`, `--brand-teal-dark`, `--brand-teal-deeper` are named from the original teal BaltimoreAI palette but now hold the *charcoal* BethesdaAI colors (`--brand-teal: #18161c`, etc.). `--brand-accent` (`#dd6b30`, burnt orange) and `--brand-accent-dark` were added during the rebrand and don't exist on pages that haven't been touched yet. Don't rename the existing variables (that would balloon every diff); just be aware the name lies about the color.

The logo mark is a serif "B" (Playfair Display, weight 900) with a small orange dot via `::after`, implemented as `<span class="logo-mark" aria-hidden="true">B</span>` next to a `Bethesda<span class="accent">AI</span>` wordmark — not an `<img>`. The old `assets/brand/logo-mark.png` (a teal Baltimore skyline/harbor icon) is stale and shouldn't be reintroduced.

`assets/brand/og-image.png` (1200×630, used for all `og:image`/`twitter:image` tags) is a **rendered screenshot**, not a hand-designed asset — its text (headline, tagline, "EST. 2023 · BETHESDA, MD" badge) is baked into the pixels, so it goes stale silently when on-page copy changes and won't show up in a text grep. Regenerate it by building a small HTML file styled like the hero section and screenshotting it at exactly 1200×630 with Playwright (`deviceScaleFactor: 1`) rather than editing pixels directly.

### Nav dropdown pattern

The top nav's "Demos" link is a hover/click dropdown (`<li class="nav-dropdown">` containing a `.nav-dropdown-toggle` button and a `.nav-dropdown-menu` list with "All Demos" and "Legal Assistant Demo" links), with matching CSS (`.nav-dropdown*` rules, inserted just before `</style>`) and a small vanilla-JS toggle script (inserted just before `</body>`) duplicated on every page. Pages without a hamburger menu (`legal.html` and the demo sub-pages) fall back to hiding all nav `<li>`s except the last on narrow viewports rather than a hamburger; `index.html` and `demos.html` have a real `.menu-toggle` hamburger for mobile.

### Cloudflare Workers back the two "Live AI" chat demos

`worker/src/worker.js` (deployed as the `baltimoreai-demos` Worker, config in `worker/wrangler.toml`) powers the chat in `demos/knowledge-assistant.html` and the eval run in `demos/eval-lab.html`. `worker/src/worker-legal.js` (deployed as `baltimoreai-legal-demo`, config in `worker/wrangler-legal.toml`) powers the case-research chat in `demos/legal.html`. They're intentionally separate Workers with separate KV namespaces and rate-limit bindings so the legal demo's usage can't exhaust the civic demo's daily cap or vice versa.

Both Workers follow the same shape: CORS-gate on an `ALLOWED_ORIGINS` allowlist, per-IP rate limit via an `unsafe.bindings` `ratelimit` binding, a KV-backed (`DEMO_LIMITS`) daily global request cap, then a streaming proxy to `https://api.anthropic.com/v1/messages` with a fixed system prompt that grounds the assistant in a small inline "document corpus" (the demo's fake source documents are hardcoded in the system prompt string, not fetched from anywhere).

The frontend talks to a Worker via a hardcoded `const WORKER_URL = 'https://<script-name>.<account-subdomain>.workers.dev';` in each demo page's `<script>` — Cloudflare Workers' `.workers.dev` URLs are deterministic from the script name and the account's fixed subdomain, so this URL doesn't need updating on every redeploy of the *same* script name, only if the script is renamed or moved to a different account.

`worker/wrangler.toml` and `worker/wrangler-legal.toml` pin real KV namespace IDs (`DEMO_LIMITS_civic`, `DEMO_LIMITS_legal`) — don't let these regress to the `PASTE_KV_NAMESPACE_ID_HERE` placeholder that shipped in an earlier version, or a fresh `wrangler deploy` will fail.

### Hosting

`CNAME` points the intended custom domain at `bethesdaai.org`, implying GitHub Pages. **As of the last check, GitHub Pages had never actually been enabled in the repo's Settings** (zero `pages-build-deployment` Actions runs ever recorded) — merging to `main` alone does not make changes live. Confirm Pages is actually turned on (Settings → Pages → Deploy from a branch → `main` → custom domain `bethesdaai.org`) before assuming a merged change is visible on the public site.

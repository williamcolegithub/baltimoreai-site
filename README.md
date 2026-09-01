# Bethesda AI website

Marketing site and live-demo pages for **Bethesda AI** (legal name Kimura Consulting LLC), served from `bethesdaai.org`.

This is a plain static site — no build step, no package manager, no framework. Every `.html` file is self-contained with its own inline `<style>` and `<script>`.

## Structure

```
index.html              Homepage
demos.html               Demo directory / filterable grid of all live demos
demos/
  legal.html             Law firm demo (case research chat, contract review, drafting)
  knowledge-assistant.html   Cited Q&A chat demo over a small document set
  eval-lab.html           Live eval run demo (score → improve → verify a RAG assistant)
  pipeline-runner.html    Simulated email-in/automation-pipeline demo
  document-processor.html Simulated document field-extraction demo
  index.html              Redirect stub to ../demos.html
privacy.html / terms.html Legacy pages for an unrelated prior product ("SmartText Assistant"); noindex
worker/
  src/worker.js           Cloudflare Worker backing the knowledge-assistant + eval-lab demos
  src/worker-legal.js     Cloudflare Worker backing the legal demo's case-research chat
  wrangler.toml / wrangler-legal.toml   Worker deploy configs
assets/brand/             Logo, OG share image
CNAME                     Custom domain for GitHub Pages (bethesdaai.org)
```

## Local development

There's no dev server. Serve the repo root with any static file server and open a page:

```
python3 -m http.server 8000
# then open http://localhost:8000/index.html
```

## Deploying the Workers

The two chat demos call out to live Cloudflare Workers, which proxy to the Anthropic API. From `worker/`:

```
wrangler secret put ANTHROPIC_API_KEY                        # one-time, for the main demos worker
wrangler deploy

wrangler secret put ANTHROPIC_API_KEY --config wrangler-legal.toml  # one-time, for the legal demo worker
wrangler deploy --config wrangler-legal.toml
```

Each demo page has the deployed Worker's `.workers.dev` URL hardcoded in its `<script>` as `WORKER_URL` — update it by hand if a Worker is ever renamed or redeployed to a different Cloudflare account.

## Hosting

The site is intended to be served via GitHub Pages with the custom domain in `CNAME`. See `CLAUDE.md` for the current status of that setup.

## For AI coding agents

See `CLAUDE.md` for architecture notes (brand palette conventions, why every page duplicates its own nav/footer, how the Worker backends are structured, and known gaps).

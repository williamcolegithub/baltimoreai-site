# baltimoreai.org

The website of BaltimoreAI, the trade name of Kimura Consulting LLC. A static site
served by GitHub Pages from the `main` branch. A push to `main` is live in about a
minute.

There is no build step, no package manager, and no framework. Every page is one
self-contained `.html` file with its own inline styles, header, and footer. A
site-wide change has to be made in each file.

## Pages

| File | What it is |
|---|---|
| `index.html` | Home page. The contact form posts to Formspree. |
| `demos.html` | Directory of the demos. |
| `demos/*.html` | The demos. `knowledge-assistant` and `eval-lab` call the live Worker. `document-processor` and `pipeline-runner` are simulated. `demos/index.html` redirects to `demos.html`. |
| `privacy.html`, `terms.html` | Privacy policy and text-message terms for the BaltimoreAI Text Assistant. Linked from every footer. Mobile carriers review these pages, so keep them accurate. |
| `text-assistant.html` | Sign-up page for the Text Assistant. Unlisted: `noindex`, not in the sitemap, not linked. The form saves nothing until the service launches. |
| `robots.txt`, `sitemap.xml`, `CNAME` | Search and domain settings. `CNAME` must stay `baltimoreai.org`. |

## Preview locally

```
python3 -m http.server 8000
# open http://localhost:8000/index.html
```

## The demo Worker

`worker/src/worker.js` is a Cloudflare Worker that the two live demos call. It checks
the request's origin, rate-limits by IP address, counts requests against a daily cap,
and forwards the text to Anthropic's API. It stores no message text.

Pushing this repository does not deploy the Worker. It is deployed separately with
`wrangler deploy` from `worker/`. `worker/wrangler.toml` still holds a placeholder for
the KV namespace id; the live Worker is configured outside this repository.

## Local files

`_local/`, `.wrangler/`, and `.vscode/` are git-ignored. This repository is public, so
drafts and anything private belong in `_local/`.

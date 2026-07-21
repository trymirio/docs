# Versive docs

Public documentation site for [docs.getversive.com](https://docs.getversive.com), built with [Mintlify](https://mintlify.com).

## Local development

```bash
npm i -g mint
cd docs
mint dev
```

The site runs at `http://localhost:3000`. `mint broken-links` checks internal links.

## Structure

- `docs.json` — site config and navigation (three tabs: Documentation, API reference, SDK)
- `index.mdx`, `getting-started/` — landing page and core concepts
- `studies/`, `ai-tests/`, `platform/` — non-technical product docs
- `api-reference/` — public v1 REST API (MDX-defined endpoints; base URL and auth are configured under `api.mdx` in `docs.json`)
- `sdk/` — `@getversive/embed` documentation

## Deploying to docs.getversive.com

One-time setup in the [Mintlify dashboard](https://app.mintlify.com):

1. **Connect the repo** — install the Mintlify GitHub app on `trymirio/versive`.
2. **Set the monorepo path** — in *Settings → Deployment → Git Settings*, enable "docs.json is in a subdirectory" and set the path to `/docs`. Set the deployment branch (e.g. `main` or `staging`).
3. **Add the custom domain** — in *Settings → Deployment → Custom domain*, add `docs.getversive.com`. The dashboard shows the DNS record to add (a CNAME for the `docs` subdomain at your DNS provider). TLS certificates are provisioned automatically once DNS propagates.

After setup, every push to the deployment branch deploys automatically; PRs get preview deployments.

## Conventions

- Non-technical pages describe the product as it exists in the app — when features change, update the matching page.
- API pages use Mintlify's MDX API components (`ParamField`, `ResponseField`, `RequestExample`). The API base URL is set once in `docs.json` (`api.mdx.server`).
- Icons: `docs.json` sets `icons.library` to `tabler`; use plain Tabler icon names (e.g. `icon="messages"`) on Cards.
- Neutral callouts use `<Callout icon="info-circle" color="#71717a">` (zinc) instead of `<Note>`/`<Info>`, so callout styling stays on brand.

# Versive docs

Public documentation site for [docs.getversive.com](https://docs.getversive.com), built with [Mintlify](https://mintlify.com).

**Source of truth: `docs/` in the `trymirio/versive` monorepo.** Docs changes ship in feature PRs there and are then synced to the public mirror repo [`trymirio/docs`](https://github.com/trymirio/docs), which Mintlify deploys. If you're reading this in `trymirio/docs`: don't edit here — changes will be overwritten by the next sync. Fix the monorepo and re-sync.

## Local development

```bash
npm i -g mint
cd docs
mint dev
```

The site runs at `http://localhost:3000`. `mint broken-links` checks internal links.

## Structure

- `docs.json` — site config and navigation (four tabs: Documentation, API reference, MCP, SDK)
- `index.mdx`, `getting-started/` — landing page and core concepts
- `studies/`, `ai-tests/`, `platform/` — non-technical product docs
- `api-reference/` — public v1 REST API (MDX-defined endpoints; base URL and auth are configured under `api.mdx` in `docs.json`)
- `mcp/` — Versive MCP server documentation (connecting AI agents)
- `sdk/` — `@getversive/embed` documentation

## Deploying to docs.getversive.com

Deployment is a two-step flow:

1. **Docs land in the monorepo.** Every feature PR that changes user-facing behavior updates the matching page under `docs/` (see the docs rule in the root `CLAUDE.md`).
2. **Sync to the public mirror.** After the docs changes reach `main`, run the `sync-public-docs` skill (`.claude/skills/sync-public-docs/` — or just ask Claude to "sync the public docs"). It copies the tracked `docs/` tree into a local checkout of `trymirio/docs`, reports the diff for review, and you commit and push to that repo's `main`. Every push to the mirror's `main` deploys automatically via Mintlify.

Mintlify itself is configured in the [Mintlify dashboard](https://app.mintlify.com): the GitHub app is installed on `trymirio/docs` with `main` as the deployment branch, and the custom domain `docs.getversive.com` is set under *Settings → Deployment → Custom domain* (a CNAME for the `docs` subdomain; TLS is provisioned automatically).

## Conventions

- Non-technical pages describe the product as it exists in the app — when features change, update the matching page.
- API pages use Mintlify's MDX API components (`ParamField`, `ResponseField`, `RequestExample`). The API base URL is set once in `docs.json` (`api.mdx.server`).
- Icons: `docs.json` sets `icons.library` to `tabler`; use plain Tabler icon names (e.g. `icon="messages"`) on Cards.
- Neutral callouts use `<Callout icon="info-circle" color="#71717a">` (zinc) instead of `<Note>`/`<Info>`, so callout styling stays on brand.

# emdash-test

Does [EmDash](https://github.com/emdash-cms/emdash) — Cloudflare's open-source, WordPress-style CMS — run on the **Cloudflare Workers free plan**, and is its admin good enough to hand to a non-technical editor?

This repository is the stock EmDash **blog template for Cloudflare** (EmDash 1.3.1, Astro 7), scaffolded unchanged except for one dependency fix, plus the measurements below. Everything was measured on 2026-10-28 on Windows 11 with Node 22.

```
npx create-emdash@1.3.1 emdash-test --template cloudflare:blog --pm npm --no-sandboxed-plugins --install --yes
```

## Findings

| Question | Answer | How it was checked |
|---|---|---|
| Does it run locally? | Yes. `npm run dev` emulates D1, R2 and KV; site at `:4321`, admin at `/_emdash/admin` | Pages and admin loaded in headless Edge |
| Admin in French? | Yes, automatically from the browser language (setup wizard, editor, menus) | [Screenshots](docs/screenshots/) |
| What does the admin offer? | Dashboard, calendar, pages, posts, media library, comments, menus, redirects, widgets, sections, categories, tags, bylines, content types, users and roles, plugins, WordPress import, settings. Editor with drafts, revisions, preview, scheduling, SEO | Admin navigation |
| Custom fields? | `string`, `text`, `url`, `number`, `integer`, `boolean`, `datetime`, `select`, `multiSelect`, `portableText`, `image`, `file`, `reference`, `json`, `slug`, `repeater`, `blocks` — a repeater takes `string`, `text`, `url`, `number`, `integer`, `boolean`, `datetime`, `select`, `image` sub-fields | `FieldType` in `emdash` 1.3.1 typings |
| How do editors log in? | Passkey first, or a link sent by email | Login page |
| Worker size vs. the free plan | **19.4 MB raw / 5.0 MB gzip.** Cloudflare's limit is now 64 MiB uncompressed on both plans, with no compressed limit — it fits | `wrangler deploy --dry-run`; [Workers limits](https://developers.cloudflare.com/workers/platform/limits/) |
| Startup vs. the 1 s limit | 38 ms of CPU, measured locally (Cloudflare's CPUs differ) | `wrangler check startup` |
| Sandboxed plugins | Need Worker Loader, a **paid-plan** feature; disabled by default (`worker_loaders` commented out in `wrangler.jsonc`) | `create-emdash --help` |
| Bindings a deploy creates | D1 (`DB`), R2 (`MEDIA`), KV (`SESSION`), Images (`IMAGES`), static assets | Dry-run output |
| Cron | One trigger every minute — uses 1 of the 5 cron triggers of a free account | `wrangler.jsonc` |
| Security audit | 7 "high" findings, all one advisory: zod < 4.6.6 can exhaust memory ([GHSA-mrhx-c7xc-m3w5](https://github.com/advisories/GHSA-mrhx-c7xc-m3w5)), pulled in by `@emdash-cms/*`. `npm audit fix` suggests a downgrade to EmDash 0.36; instead `"overrides": { "zod": "^4.6.6" }` in `package.json` gives **0 vulnerabilities** and the build still passes | `npm audit`, `npm run build` |
| Dev shortcuts | `/_emdash/api/setup/dev-bypass` and `/_emdash/api/auth/dev-bypass` finish setup and log in as "Dev Admin" — only when `import.meta.env.DEV`, so not in a deployed build | Route source in `node_modules/emdash` |

## Not measured yet

- **CPU per request on Cloudflare** — the free plan allows 10 ms; only a deployed Worker shows it (`wrangler tail` reports CPU time per request).
- **R2 on an account without a payment card** — R2 asks for one even within its free allowance.
- **The Images binding** on the free plan.
- **Login links by email** — which email provider EmDash needs on Cloudflare.

## Run it

```
npm install
npm run dev
```

Then open http://localhost:4321/_emdash/api/auth/dev-bypass?redirect=/_emdash/admin to skip the setup wizard and land in the admin as "Dev Admin" (development only). `npm run dev` writes nothing to Cloudflare. `EMDASH_ENCRYPTION_KEY` lives in `.env`, which Git ignores.

## The template

A blog with posts, pages, categories, tags, full-text search, RSS, SEO metadata and dark mode — see EmDash's [blog template](https://github.com/emdash-cms/templates/tree/main/blog-cloudflare). `AGENTS.md` and `.agents/skills/` come with it, for AI coding assistants; `.mcp.json` points them to EmDash's documentation server.

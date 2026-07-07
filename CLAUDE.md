# JetAvenue.net

**Static Astro marketing site. There is no backend in this repo.**

Astro + `@astrojs/sitemap`, static build (no adapter, no `output: 'server'`, no `src/pages/api/`). Deploys on Vercel (project `jet-avenue`, production branch `master`). No environment variables — this project has none and needs none.

## The one thing that will bite you

The forms here POST **cross-origin** to RecoilMedia.com. The whole store → PDF → email chain lives there, not here:

- Work order form → `https://www.recoilmedia.com/api/forms/work-order`
- Contact form → `https://www.recoilmedia.com/api/forms/inquiry`

**Load-bearing, do not break:** in `src/pages/work-order.astro` (and `contact.astro`) the form's absolute `action` URL and the hidden `<input name="brand" value="jetavenue.net">` are how the RecoilMedia endpoint routes and stores the submission. Change either and submissions silently break or misroute. The absolute URL is deliberate — the form must work regardless of what domain serves the page.

**Depends on RecoilMedia.com being live.** If recoilmedia.com is down or its `/api/forms/*` routes change, JetAvenue's forms stop working even though this site is up. This repo can't fix a broken form on its own.

## Run / build / test

- `npm run dev` — local dev
- `npm run build` — static build to `dist/`
- No backend to run or mock. To verify forms end-to-end you exercise RecoilMedia.com's endpoint, not this repo (and note: that endpoint has no test mode and hardcoded real recipients — see `docs/architecture.md`).

More detail, including the full submission flow and the transfer/deploy notes: [`docs/architecture.md`](docs/architecture.md).

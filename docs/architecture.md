# JetAvenue architecture

Written from the pre-transfer audit (2026-07-07). The point of this document: **the JetAvenue repo is only a static marketing site; the "critical app" behavior everyone associates with JetAvenue actually runs on RecoilMedia.com.**

## What this repo is

- **Framework:** Astro (static). Dependencies are just `astro` + `@astrojs/sitemap`. No SSR adapter, no `output: 'server'`, no `src/pages/api/` routes, no serverless functions in the build.
- **Deploy:** Vercel project `jet-avenue`, framework preset `astro`, production branch `master`. **Zero environment variables** on the project — it needs none, because it does no server-side work.
- **Domains:** `jetavenue.net` (redirects to www) and `www.jetavenue.net`.
- **Pages:** marketing pages (index, about, services, contact, faq, work-order, legal) plus `contact/thank-you` and `work-order/thank-you` landing pages the form flow redirects to.

## The submission flow (lives on RecoilMedia.com, not here)

The work-order form (`src/pages/work-order.astro`) is plain HTML that does a native multipart POST:

```
action="https://www.recoilmedia.com/api/forms/work-order"  method=POST  enctype=multipart/form-data
hidden field: brand = "jetavenue.net"
```

On submit it compresses photos client-side, native-POSTs, and the endpoint 303-redirects the visitor to `https://www.jetavenue.net/work-order/thank-you`. The contact form works the same way against `/api/forms/inquiry`.

Everything downstream is **RecoilMedia.com's `src/pages/api/forms/work-order.ts`** and its Vercel project `recoil-media` (which holds `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `RESEND_API_KEY`, `ADMIN_EMAILS`, etc.):

1. **Store** — inserts one row into Supabase `form_submissions` (service-role key). Best-effort resolves Organization > Client > Brand from the brand's website.
2. **Uploads** — compresses + stores the customer's file uploads to Supabase Storage under `jetavenue.net/<row-id>/`.
3. **PDF** — generated **in-app at runtime** (`buildWorkAuthorizationPdf` + `buildCreditCardAuthPdf` in `src/lib/pdf/workOrderDocs`), not an external service. The generated PDFs are stored on the row too.
4. **Email** — via **Resend**, FROM `no-reply@mail.recoilmedia.com`, sent **1-to-1** (never CC/BCC — batched sends were spam-filtered).
   - Admin notification recipients are **hardcoded in the endpoint**, not read from form data: `mx@jetavenue.net` plus `admin@recoilmedia.com` (platform archive copy).
   - Customer copy goes to the email the submitter typed into the form (reply-to `mx@jetavenue.net`), with both PDFs attached.
5. Row `status` is flipped to `emailed` on success.

There is **no test mode / dry-run** on that endpoint, and the admin recipients are real inboxes. Any real submission with `brand=jetavenue.net` emails those inboxes and writes real data — see "Testing" below.

## What depends on what

- JetAvenue's forms depend on **RecoilMedia.com being live** and its `/api/forms/*` routes staying stable. If recoilmedia.com is down, JetAvenue's forms break even though this site is up.
- The absolute `action` URL means the form is **origin-independent** — it works no matter what domain (or Vercel preview) serves the JetAvenue page. Don't "fix" it into a relative path.
- No Supabase↔GitHub integration, no branching/migration automation tied to either repo. RecoilMedia's DB migrations are raw `.sql` files applied manually.
- The only cron in the system (`/api/cron/review-ratings`, daily) runs on `recoil-media`, unrelated to JetAvenue. JetAvenue has none.

## GitHub-org transfer notes (2026-07-07)

Transferred `AnchorisLLC/JetAvenue` → `vevric/JetAvenue` (repo id preserved: `1263427006`). What that transfer could and couldn't touch:

- **At risk (repo-tied):** only auto-deploy of this static site. Like every project moved into vevric, the Vercel Git connection goes stale on transfer (link metadata follows, credential doesn't) and must be **reconnected once** in the Vercel dashboard (Settings → Git → Disconnect → Connect `vevric/JetAvenue`). The live site stays up on its last build until then.
- **Not at risk:** no GitHub Actions, secrets, webhooks, deploy keys, or branch protection existed on this repo. The store → PDF → email chain is untouched because it lives on RecoilMedia.com, and the form keeps working throughout because it posts to the absolute recoilmedia.com URL.

## Testing

- **Static site / deploy:** safe to verify freely — fetch the live pages, confirm 200, and confirm the served work-order HTML still carries `action="https://www.recoilmedia.com/api/forms/work-order"` and the `brand="jetavenue.net"` hidden field. Zero emails, zero data.
- **Full form → store → PDF → email chain:** there is **no fully safe live test**. Recipients are hardcoded real inboxes and there's no test mode, so a real submission emails `mx@jetavenue.net` + `admin@recoilmedia.com` and writes real data (unrecallable email, deletable row). Exercising that chain safely requires a temporary test-brand route added on **RecoilMedia.com** — which is a RecoilMedia change, not a JetAvenue one.

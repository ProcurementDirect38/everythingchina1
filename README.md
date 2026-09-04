# everythingchina.com.au

Static marketing website for **Everything China** — an Australian-based China sourcing,
procurement, quality control, logistics and supply-chain management company.

Zero dependencies. Content lives in JSON, components live in one Python module, and the
build produces plain HTML/CSS that will deploy to any static host.

---

## Requirements

Python 3.8+. Nothing else — no Node, no `npm install`, no package manager.

## Build & run

```bash
python3 build.py            # build into dist/
python3 build.py --serve    # build, then serve dist/ at http://localhost:8000
```

The build prints a warning if the enquiry form has no endpoint configured, and a count of
outstanding content placeholders.

## Deploy

`dist/` is the deployable artefact. Upload it to any static host (Netlify, Cloudflare Pages,
S3 + CloudFront, Vercel, or plain nginx). No server-side runtime is required for the pages
themselves — only the enquiry form needs a backend (see below).

Configure the host to serve `dist/` as the site root. Directory-style URLs (`/services/`)
resolve to `index.html` automatically on all the hosts above.

---

## Project layout

```
build.py                  Static site generator (stdlib only)
src/templates.py          Every reusable component; one function per section type
src/styles/tokens.css     THE ONLY FILE THAT DEFINES COLOUR — see "Branding" below
src/styles/main.css       Layout and components; consumes tokens, defines no colour
src/js/site.js            Mobile nav + sticky CTA (progressive enhancement only)
src/js/form.js            Enquiry form validation and submission
src/js/submit-handler.js  THE ONLY FILE THAT REFERENCES THE BACKEND
content/site.json         Nav, footer, CTAs, contact details
content/faqs.json         FAQ questions and answers (shared across pages)
content/<page>.json       One file per page: metadata + an ordered list of sections
static/                   Files copied verbatim into dist/
dist/                     Build output — generated, safe to delete
```

### Editing copy

All text lives in `content/*.json`. Change the copy, run `python3 build.py`, done — no
component code needs touching. Each page file is:

```json
{
  "meta": { "path": "/services/", "title": "…", "description": "…" },
  "sections": [ { "type": "cards", "heading": "…", "items": [ … ] } ]
}
```

`type` selects a renderer from `RENDERERS` in `src/templates.py`. The available types are
`hero`, `pageHero`, `cards`, `chips`, `steps`, `categories`, `featureSplit`, `panels`,
`processGrid`, `flow`, `cases`, `faq`, `prose`, `services`, `sourceGroups`,
`contactDetails`, `form` and `ctaBand`. Optional keys on any section: `id` (anchor target)
and `bg` (`"alt"` for the tinted background, `"invert"` for the dark band).

An unknown `type` fails the build loudly rather than silently rendering nothing.

### Adding a page

1. Create `content/<name>.json` with a `meta.path` and `sections`.
2. Add `"<name>"` to `PAGES` in `build.py`.
3. Add it to `nav` or `footer` in `content/site.json` if it should be linked.

It is picked up by the sitemap automatically.

---

## Branding

**Colour.** Every colour value in the project lives in `src/styles/tokens.css`. The values
currently in place are **neutral placeholders, not a brand palette**. To apply a real
palette, edit the hex values in the `COLOUR` block of that file and rebuild — nothing else
needs to change. `main.css` contains no literal colour values, and the build has been
verified to keep it that way.

If you change the palette, re-run the contrast check before shipping. Every text/background
pair currently meets WCAG AA (4.5:1 for text, 3:1 for the focus ring); the tightest pair is
`--color-ink-subtle` on `--color-surface-sunken` at 4.53:1, so darken that token if you
lighten the sunken surface.

**Logo.** There is deliberately no logo. The header and footer use a text wordmark built
from `wordmark` in `content/site.json`. To add a logo later, replace the contents of the
`.wordmark` anchor in `header()` / `footer()` in `src/templates.py` with an `<img>`; the
sizing and alignment hooks are already in `.wordmark` in `main.css`.

**Typography.** A system font stack, for zero network requests and no layout shift. To use
a brand typeface, self-host the font files, add `@font-face` rules with
`font-display: swap`, and change `--font-sans` / `--font-display` in `tokens.css`.

---

## Connecting the enquiry form

The form posts `multipart/form-data` because it carries file uploads. **`src/js/submit-handler.js`
is the only file that references a backend.**

1. Set `ENDPOINT` in that file to your receiving URL — your own API route, a serverless
   function, or a hosted form provider. Anything that accepts a `POST` of
   `multipart/form-data` and returns 2xx on success will work.
2. If the provider needs auth headers, add them in `buildRequest`.
3. Set the same URL as the `action` in `content/contact.json` so the no-JavaScript path
   posts to the same place.

While `ENDPOINT` is empty the form falls back to a native browser POST to the form `action`,
so nothing is silently swallowed — but a static host cannot accept that POST. **Wire the
endpoint before launch.** The build warns you until you do.

### What the server must do

Client-side validation is a convenience, not a control. The receiving endpoint must repeat
every check independently:

- Required fields: `name`, `company`, `email`, `sourcing`, `description`.
- File uploads: at most 10 files, 10 MB each, extensions limited to
  `pdf, jpg, jpeg, png, docx, xlsx, csv, dwg`. Validate the actual content type, not just
  the filename, and store uploads outside the web root.
- Spam traps: reject any submission where the hidden `website` field is non-empty.
- Rate-limit by IP.

---

## Accessibility

Verified in this build: one `<h1>` per page, no skipped heading levels, landmark regions
(`header` / `nav` / `main` / `footer`), a skip link, visible focus rings, an accessible
mobile nav (`aria-expanded` + `aria-controls`, Escape to close, focus returned to the
trigger), real form labels with `aria-describedby` error slots announced via `role="alert"`,
and `prefers-reduced-motion` respected on every transition.

The FAQ uses native `<details>`/`<summary>`, so it works with JavaScript disabled.

## Performance notes

- No external requests: no CDN, no web fonts, no analytics, no third-party scripts.
- CSS and JS are versioned by content hash (`main.css?v=…`), so they can be cached
  immutably and still update instantly on deploy.
- `site.js` is `defer`; `form.js` loads only on the contact page.
- The site has no images yet. When you add them, use `srcset`, modern formats, explicit
  `width`/`height` to prevent layout shift, and `loading="lazy"` below the fold.

## SEO

Per-page `<title>` and meta description, canonical URLs, Open Graph and Twitter tags, plus
JSON-LD for `Organization` (all pages), `BreadcrumbList` (inner pages), `FAQPage` (pages
with an FAQ) and an `ItemList` of `Service` entries on the services page.
`sitemap.xml` and `robots.txt` are generated on every build.

---

## Before launch

See `CONTENT-TODO.md` for the full list of outstanding content. The blocking items are
contact details, the ABN, and the form endpoint.

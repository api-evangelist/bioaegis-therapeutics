---
name: Track BioAegis Therapeutics press releases and regulatory milestones
description: Build and incrementally poll an index of BioAegis Therapeutics' 94 News press releases and 34 Publications over the anonymous WordPress REST content API — covering the BTI-203 Phase 2 ARDS trial, the FDA Fast Track designations for ARDS and decompression sickness, the BARDA and U.S. Navy contracts, financings and board appointments — including how to detect change on a surface that serves no ETag or Last-Modified.
api: openapi/bioaegis-therapeutics-content-openapi.yml
base_url: https://www.bioaegistherapeutics.com/wp-json
operations:
  - listCategories
  - listPosts
  - getPost
  - searchContent
generated: '2026-08-07'
method: generated
---

# Track BioAegis Therapeutics press releases and regulatory milestones

BioAegis Therapeutics is a privately held, clinical-stage biopharmaceutical company. It files no
10-Ks and issues no earnings calls, so **its press stream is the primary public record of its
clinical and regulatory progress** — FDA Fast Track designations, IND clearances, trial site
activation, BARDA and U.S. Navy contract awards, financings and board changes.

That stream is fully retrievable over the site's own WordPress REST API without credentials.
153 posts were registered at harvest time (`X-WP-Total: 153` on 2026-08-07), of which 94 are in the
News category and 34 in Publications.

## Before you start

- **No credentials.** `GET /wp/v2/posts` returns 200 anonymously.
- **Always project.** `content.rendered` is populated on every post. An unfiltered
  `per_page=100` request returns megabytes. Send `_fields`.
- **`author` does not resolve.** `/wp/v2/users` returns **403 as an HTML page** from the Sucuri
  CloudProxy WAF — not a JSON error. Treat `author` as an opaque integer and branch on
  `Content-Type` before parsing any error body.
- **There are no tags.** `Post.tags` is an empty array on every record. Category is the only
  classification axis.

## Steps

### 1. Resolve the category IDs first — do not hardcode them

Call `listCategories` (`GET /wp/v2/categories`):

```
GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count
```

Observed on 2026-08-07 (10 terms):

| id | slug | name | count |
|----|------|------|-------|
| 11 | news | News | 94 |
| 12 | publication | Publication | 34 |
| 15 | event | Event | 9 |
| 5 | clinical | Clinical | 6 |
| 7 | leadership | Leadership | 5 |
| 25 | board | Board | 4 |
| 24 | feature | Feature | 2 |
| 14 | covid19 | COVID19 | 1 |
| 1 | uncategorized | Uncategorized | 1 |
| 17 | jobs | Jobs | 0 |

Match on `slug`, not on the integer — WordPress term IDs are stable in practice but nothing
documents them, because nothing about this surface is documented.

### 2. Build the press index

Call `listPosts` (`GET /wp/v2/posts`) filtered to the News category:

```
GET /wp/v2/posts?categories=11&per_page=100&page=1&orderby=date&order=desc&_fields=id,slug,title,link,date,modified,excerpt,categories,featured_media
```

Read `X-WP-Total` and `X-WP-TotalPages` from the response headers rather than assuming — the body
is a bare JSON array with no envelope, so totals exist **only** in headers. Follow
`Link: rel="next"`. `per_page` outside 1-100 returns `400 rest_invalid_param` with the offending
parameter named in `data.params`.

Swap `categories=12` for the Publications stream, `categories=15` for Events.

### 3. Pull a body when you need one

Call `getPost` (`GET /wp/v2/posts/{id}`) for a single release:

```
GET /wp/v2/posts/5904
```

`content.rendered` carries the full press release as HTML (the April 2026 Prenosis collaboration
announcement is ~9KB). `excerpt.rendered` is a usable summary if you only need the gist —
prefer it, and only fetch the body when the excerpt is insufficient.

A non-existent or unpublished ID returns `404 rest_post_invalid_id`. The API does not distinguish
"never existed" from "not published", so do not infer deletion from a 404.

### 4. Find a milestone by name

Call `searchContent` (`GET /wp/v2/search`) when you know the topic but not the ID:

```
GET /wp/v2/search?search=fast+track&type=post&subtype=any&per_page=20
```

A search for `gelsolin` returns 137 matches across posts and pages. Results are lightweight
`{id, title, url, type, subtype}` records — resolve `id` against `getPost` (or `getPage` when
`subtype` is `page`) for the body.

### 5. Poll for change

There is **no ETag, no Last-Modified and no Cache-Control** on this surface, so conditional
requests are impossible and you cannot ask "has this changed" cheaply over HTTP.

Two workable signals, in order of cost:

1. **Sitemap lastmod (cheapest).** `GET https://www.bioaegistherapeutics.com/sitemap_index.xml`
   returns per-child `lastmod` values. If `post-sitemap.xml` has not moved, nothing was published.
2. **Newest-modified probe.** One tiny call:

```
GET /wp/v2/posts?_fields=id,modified&orderby=modified&order=desc&per_page=1
```

Compare `modified` against your stored high-water mark; fetch the delta with
`?modified_after=` or `?after=` only when it has advanced.

Do not poll aggressively. The origin sits behind a Sucuri CloudProxy WAF that publishes **no rate
limit** — no `RateLimit`, no `X-RateLimit-*`, no `Retry-After` — and may throttle or challenge
without advertising a budget. Hourly is more than enough for a company that publishes a few times
a quarter.

## What you will find in the archive

The News stream runs from 2011 (the exclusive option on the Brigham and Women's Hospital plasma
gelsolin license) to April 2026. High-signal entries include the FDA Fast Track designations for
ARDS and for inflammasome-driven decompression sickness, the $20M BARDA DRIVe contract
(75A50123C00067), the U.S. Navy Office of Naval Research contract via the University of Maryland
School of Medicine, IND clearance for ARDS, site activation across 13 countries for the
600-patient BTI-203 Phase 2 trial, the $22M institutional equity sale, and the Prenosis
AI-precision-medicine collaboration.

## Related artifacts

- `conventions/bioaegis-therapeutics-conventions.yml` — pagination, projection, caching, rate-limit posture
- `errors/bioaegis-therapeutics-problem-types.yml` — every error code, provoked live
- `lifecycle/bioaegis-therapeutics-lifecycle.yml` — why there is no changelog to subscribe to

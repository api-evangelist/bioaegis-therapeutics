---
name: Map the BioAegis Therapeutics leadership, board, clinical advisors and publication record
description: Retrieve BioAegis Therapeutics' people graph and scientific citation record over the anonymous WordPress REST content API — 5 leadership profiles, 4 corporate board members, 6 clinical advisory board members and 34 publication entries, all authored as categorized posts rather than as a custom post type — plus the 247-item media library that holds their headshots and the company's scientific figures.
api: openapi/bioaegis-therapeutics-content-openapi.yml
base_url: https://www.bioaegistherapeutics.com/wp-json
operations:
  - listCategories
  - listPosts
  - getPost
  - listMedia
  - getMediaItem
  - searchContent
generated: '2026-08-07'
method: generated
---

# Map the BioAegis Therapeutics leadership, board, clinical advisors and publication record

For a clinical-stage company, the people and the peer-reviewed record *are* the diligence surface.
BioAegis publishes both, and — unusually — publishes them through the **post** collection rather
than through a custom post type, segmented only by category. A consumer looking for a `/people` or
`/team` endpoint will not find one and may wrongly conclude the data is absent.

## Before you start

- **No credentials.** All operations below return 200 anonymously.
- **People are posts.** There is no `team`, `person` or `leadership` post type. `GET /wp/v2/types`
  registers 13 types; only `post`, `page` and `attachment` hold content.
- **`author` is a dead field.** `/wp/v2/users` returns 403 as HTML from the Sucuri WAF. The `author`
  integer on each profile post is the CMS user who typed it, not the person the profile is about.
  Do not confuse the two.
- **Project.** `content.rendered` is populated; send `_fields`.

## Steps

### 1. Resolve the people categories

Call `listCategories` (`GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count`).

Three categories carry people, observed 2026-08-07:

| id | slug | name | count | who |
|----|------|------|-------|-----|
| 7 | leadership | Leadership | 5 | executive team |
| 25 | board | Board | 4 | corporate board of directors |
| 5 | clinical | Clinical | 6 | clinical advisory board |

Match on `slug`. A fourth, `jobs` (17), is registered with 0 posts — the Careers page carries the
recruiting narrative instead.

### 2. Pull each cohort

Call `listPosts` (`GET /wp/v2/posts`) per category:

```
GET /wp/v2/posts?categories=7&per_page=100&_fields=id,slug,title,link,excerpt,featured_media,modified
```

The post `title` is the person's name and credentials (for example
`Howard Levy, M.B.B.Ch., Ph.D., M.M.M.` — Chief Medical Officer, and
`Susan Levinson, Ph.D.` — Chief Executive Officer). The permalink structure mirrors the category:
profiles live under `/leadership/`, `/board/` and `/clinical/` respectively, which is a useful
cross-check that you have the right cohort.

`excerpt.rendered` is usually the person's role line; `content.rendered` (via `getPost`) carries
the full biography.

### 3. Pull the publication record

Category 12 (`publication`) holds 34 entries — the peer-reviewed literature the company cites for
plasma gelsolin, spanning the New England Journal of Medicine, Clinical Infectious Diseases, the
Journal of Infectious Diseases and others.

```
GET /wp/v2/posts?categories=12&per_page=100&orderby=date&order=desc&_fields=id,title,link,date,excerpt
```

These are citation *entries*, not the papers themselves — expect a title, a journal reference and
an outbound link in the body. Fetch `getPost` for the body when you need the DOI or journal URL.

Page 2005 (`publications-plasma-gelsolin`) is the human-facing index of the same material.

### 4. Get the headshots and scientific figures

Call `listMedia` (`GET /wp/v2/media`). 247 attachments were registered at harvest time
(`X-WP-Total: 247` on 2026-08-07), predominantly WebP.

```
GET /wp/v2/media?per_page=100&page=1&media_type=image&_fields=id,slug,title,alt_text,caption,mime_type,source_url,post,date,modified
```

Read `X-WP-Total` and `X-WP-TotalPages` from the headers and follow `Link: rel="next"` rather than
assuming a page count.

To go from a person to their headshot, take `featured_media` from the profile post and call
`getMediaItem` (`GET /wp/v2/media/{id}`) — or add `_embed` to the post request and read
`_embedded['wp:featuredmedia']`. Note that `_embed` will ALSO try to embed the author relation,
which fails on this deployment; expect an error stub in `_embedded.author` and ignore it.

`media_details.sizes` gives you pre-rendered variants (thumbnail, medium, large) so you rarely need
to resize anything yourself. Attachment 2910 is the company logo referenced from the REST index
`site_logo` field.

Assets are served from `https://www.bioaegistherapeutics.com/wp-content/uploads/YYYY/MM/...`.
Respect the company's copyright — this skill retrieves metadata and URLs, it does not license the
imagery or the headshots.

### 5. Cross-reference a name across the whole site

Call `searchContent` (`GET /wp/v2/search`) to find every mention of a person across posts and
pages at once:

```
GET /wp/v2/search?search=Stossel&type=post&subtype=any&per_page=20
```

This is how you connect a profile to the press releases that announce the appointment, the
publications that cite the work, and the events where the person presented. Dr. Thomas P. Stossel,
the discoverer of gelsolin and the company's founding scientist, appears across all three.

## Caveats worth carrying downstream

- **Counts are as-of.** The category `count` field is authoritative at read time; do not cache it.
- **No tags.** `Post.tags` is empty on every record — there is no secondary axis to pivot on.
- **PII posture.** These are public corporate biographies the company chose to publish. Treat them
  as such; do not enrich them against external identity sources.

## Related artifacts

- `data-model/bioaegis-therapeutics-data-model.yml` — the full entity graph, including why `author` dangles
- `errors/bioaegis-therapeutics-problem-types.yml` — the HTML-not-JSON 403 you will hit if you probe `/wp/v2/users`

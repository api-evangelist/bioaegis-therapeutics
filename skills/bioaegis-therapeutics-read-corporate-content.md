---
name: Read BioAegis Therapeutics corporate, scientific and clinical-trial content
description: Enumerate BioAegis Therapeutics' 14 published pages — About Us, Our Science, Our Platform, BTI-203 Clinical Trial, FAQs, Publications, Careers, Contact Us, Expanded Access Policy, Code of Ethics/Conduct, Terms of Use, Privacy Policy — over the anonymous WordPress REST content API, and pull the company's schema.org identity graph, without accidentally downloading megabytes of Elementor markup.
api: openapi/bioaegis-therapeutics-content-openapi.yml
base_url: https://www.bioaegistherapeutics.com/wp-json
operations:
  - getApiIndex
  - listTypes
  - listPages
  - getPage
  - getYoastHead
generated: '2026-08-07'
method: generated
---

# Read BioAegis Therapeutics corporate, scientific and clinical-trial content

The corporate narrative — what the company is, what plasma gelsolin does, what BTI-203 is testing
and in whom — lives in 14 WordPress pages, all readable anonymously over the site's own REST API.

Unlike many WordPress corporate sites, **the bodies are actually there**: `content.rendered` and
`excerpt.rendered` are populated on every page. That is good news and a trap at the same time.

## Before you start

- **No credentials.** `GET /wp/v2/pages` returns 200 anonymously.
- **The page bodies are huge.** They are rendered Elementor markup. Page 5753 (Our Platform)
  returns roughly **100KB** in `content.rendered` alone; page 5277 (Our Science) roughly 45KB. An
  unfiltered `GET /wp/v2/pages?per_page=100` is a multi-megabyte response for 14 documents.
  **Always send `_fields`** and pull bodies one at a time, deliberately.
- **The page tree is flat.** `parent` is 0 on all 14 pages. There is no hierarchy to walk.
- **`author` does not resolve.** `/wp/v2/users` is blocked at the edge by the Sucuri WAF and
  answers **403 in HTML**, not JSON. Branch on `Content-Type` before parsing an error.

## Steps

### 1. Confirm the surface before you crawl it

Call `getApiIndex` (`GET /wp-json/`) once:

```
GET /wp-json/?_fields=name,description,namespaces,authentication,site_logo,page_on_front
```

This returns the site name (`BioAegis Therapeutics`), tagline (`The Power of Innate Immunity`), the
17 registered namespaces, the advertised authentication providers, the site logo attachment ID
(2910) and the front-page ID (901). It is also the **only** way to detect that the contract
changed: 13 of the 17 namespaces are plugin- or theme-owned and will appear and disappear with
plugin upgrades.

Optionally call `listTypes` (`GET /wp/v2/types`) to confirm which post types carry public content.
Thirteen are registered; only `post`, `page` and `attachment` hold anything on this deployment.

### 2. Enumerate the pages — metadata only

Call `listPages` (`GET /wp/v2/pages`) with a tight projection:

```
GET /wp/v2/pages?per_page=100&orderby=title&order=asc&_fields=id,slug,title,link,date,modified,excerpt
```

Observed on 2026-08-07 (14 pages, `X-WP-Total: 14`):

| id | slug | page |
|----|------|------|
| 901 | home | Home |
| 1023 | about-us | About Us |
| 5277 | our-science | Our Science |
| 5753 | our-platform | Our Platform |
| 4401 | bti-203-clinical-trial | BTI-203 Clinical Trial |
| 3585 | faqs | FAQs |
| 2005 | publications-plasma-gelsolin | Publications |
| 903 | news-plasma-gelsolin | News |
| 902 | careers | Careers |
| 5561 | contact-us | Contact Us |
| 4678 | expanded-access-policy | Expanded Access Policy |
| 4020 | code-of-ethics-conduct | Code of Ethics/Conduct |
| 485 | terms-of-use | Terms of Use |
| 3 | privacy-policy | Privacy Policy |

`excerpt.rendered` is a short, clean prose summary of each page and is usually all you need —
it is a few hundred bytes against a body that may be a hundred kilobytes.

You can also address a page by slug rather than by ID, which is more durable:

```
GET /wp/v2/pages?slug=bti-203-clinical-trial&_fields=id,title,content
```

### 3. Pull one body at a time

Call `getPage` (`GET /wp/v2/pages/{id}`) only for the pages whose full prose you actually need:

```
GET /wp/v2/pages/4401
```

Page 4401 (BTI-203 Clinical Trial) carries the trial design in prose: a randomized, double-blind,
placebo-controlled Phase 2 proof-of-concept study of rhu-pGSN added to standard of care in
moderate-to-severe ARDS (P/F ratio <= 150) caused by infection, primary endpoint survival without
organ failure at Day 28, 600 subjects across 75 sites in 13 countries, supported by BARDA contract
75A50123C00067.

Page 5277 (Our Science) and 5753 (Our Platform) carry the mechanism-of-action narrative. Page 3585
(FAQs) is the most citation-dense plain-language explanation of pGSN on the site.

Strip the Elementor wrapper markup before feeding any of it downstream — the useful text is a
small fraction of the bytes.

### 4. Take structured facts from the schema.org graph, not from the prose

Call `getYoastHead` (`GET /yoast/v1/get_head`):

```
GET /yoast/v1/get_head?url=https%3A%2F%2Fwww.bioaegistherapeutics.com%2F
```

`json.schema['@graph']` returns the site's schema.org JSON-LD — five nodes: WebPage, ImageObject,
BreadcrumbList, WebSite and Organization. `json` also carries the canonical URL, robots
directives, Open Graph and Twitter card metadata.

**Know its limits before you rely on it.** The Organization node on this site is thin: `name` and
`url` only, with a logo ImageObject whose `url` and `contentUrl` are both **empty strings**, no
`legalName`, no `address`, no `foundingDate` and no `sameAs`. If you need the legal entity, the
North Brunswick address or the social profiles, they are in the page HTML footer and in this
repository's `apis.yml` — not in the graph.

This operation also exists **only** because the Yoast SEO plugin is installed. It is the richest
structured-data source on the surface and the most fragile thing on it.

### 5. Governance documents

Pages 4678 (Expanded Access Policy), 4020 (Code of Ethics/Conduct), 485 (Terms of Use) and 3
(Privacy Policy) are the company's published policy set. The Expanded Access Policy is the one
that matters clinically — it states BioAegis' position on pre-approval access to rhu-pGSN. Note
the Terms of Use and Privacy Policy both carry a "Last Updated: December 11, 2015" line in their
body text while their WordPress `modified` timestamps read 2025-12-11; trust the body text for
policy vintage, not the CMS timestamp.

## Related artifacts

- `json-ld/bioaegis-therapeutics-organization.jsonld` — the provider's own graph, saved verbatim
- `data-model/bioaegis-therapeutics-data-model.yml` — the entity graph and its one broken arc
- `conventions/bioaegis-therapeutics-conventions.yml` — why `_fields` matters so much here

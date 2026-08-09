---
name: Read Centrexion's corporate content
description: >-
  Retrieve Centrexion Therapeutics' public corporate pages — pipeline, leadership, contact and
  policies — from the WordPress REST API behind centrexion.com, including the rendered HTML that is
  the only machine-reachable form of its drug-programme and staff content.
api: openapi/centrexion-therapeutics-content-openapi.yml
operations: [listPages, getPage, search, getApiIndex]
---

# Read Centrexion's corporate content

## What this surface actually is

Centrexion Therapeutics is a late clinical-stage biopharmaceutical company in Boston developing
non-opioid therapies for chronic pain. It **runs no developer programme and markets no API**. What
you are calling is the WordPress CMS behind its marketing site, which happens to be public and
machine-readable. There is no support channel, no SLA and no compatibility promise — see
`lifecycle/centrexion-therapeutics-lifecycle.yml`. Treat every response as best-effort.

Base URL: `https://centrexion.com/wp-json`. No credentials. No API key. Nothing to sign up for.

## Set expectations before you start

Six pages is the entire public site. Specifically:

| id | slug | page |
|----|------|------|
| 15 | `home` | Home |
| 16 | `team` | Team |
| 17 | `pipeline` | Pipeline |
| 18 | `contact` | Contact |
| 174 | `terms-of-use` | Terms of Use |
| 178 | `privacy-policy` | Privacy Policy |

Do **not** go looking for a news archive here. `listPosts` returns exactly one item: the stock
WordPress "Hello world!" placeholder. Centrexion's press releases are not on this API.

## Steps

### 1. Confirm the surface is still there

Call `getApiIndex` (`GET /`). Check that `wp/v2` appears in `namespaces`. This costs one request and
tells you whether the contract you are about to rely on still exists — worth doing because nothing
here is versioned in practice.

### 2. List the pages

Call `listPages` (`GET /wp/v2/pages`).

- Use `per_page=100` so you get all six in one request. The maximum is 100; anything higher returns
  `400 rest_invalid_param`.
- Use `_fields=id,slug,title,link,modified` first if you only need the index. Fetching full page
  bodies pulls a lot of rendered HTML you may not want.
- Do **not** pass `status`. It defaults to `publish`, and anonymously any other value returns
  `400 rest_invalid_param` rather than an empty set.

### 3. Fetch the page you actually want

Call `getPage` (`GET /wp/v2/pages/{id}`) with the id from step 2.

The substance is in `content.rendered` as HTML. This matters:

- **The drug pipeline is not data.** `pages/17` renders CNTX-4975 and the immunology programmes as
  HTML. There is no structured representation of programmes, indications, mechanisms or clinical
  phases anywhere on this API. If you need those as fields, you must parse the HTML, and you should
  say so rather than implying the API returned them structured.
- **The staff biographies are not on this API at all.** `pages/16` renders 14 leadership bios from a
  WordPress custom post type that is **not** REST-registered — it does not appear in `listPostTypes`
  and has no route. The only machine-reachable form is the HTML inside `content.rendered`.

### 4. Or search across everything

Call `search` (`GET /wp/v2/search?search=<term>`). Seven objects are searchable — the six pages plus
the placeholder post. You get back lightweight hits (`id`, `title`, `url`, `type`, `subtype`); follow
`id` into `getPage` for the body.

## Conventions that will bite you

- **Trim responses with `_fields`.** Verified working. `_embed` also works and inlines `author`,
  `replies` and `wp:term`.
- **Cache is up to 10 minutes stale.** Responses are WP Engine edge-cached
  (`cache-control: max-age=600`, `x-cacheable: SHORT`). There is **no ETag and no Last-Modified**, so
  you cannot make a conditional request — do not build a revalidation loop, it will just re-download.
- **Pace yourself.** No rate-limit headers are published, but `robots.txt` asks for `Crawl-delay: 10`.
  Honour it. Absence of a published limit is not absence of a limit.
- **Branch on `code`, never on `message`.** Errors return
  `{"code": ..., "message": ..., "data": {"status": ...}}` — the WordPress envelope, *not* RFC 9457
  problem+json. Full catalogue in `errors/centrexion-therapeutics-problem-types.yml`.

## Errors you will actually hit

| Status | `code` | Meaning |
|---|---|---|
| 404 | `rest_post_invalid_id` | That page id does not exist or is not published. Re-resolve from `listPages`. |
| 400 | `rest_invalid_param` | `per_page` above 100, or you passed a `status` other than `publish`. |
| 404 | `rest_no_route` | Wrong path, or you tried a method other than GET. The public surface is GET-only. |

## Do not

- Do not attempt writes. The entire public surface is read-only; write routes require WordPress
  application passwords, which you do not have and should not seek.
- Do not walk `/wp/v2/users`. It returns records about real people anonymously. It is documented in
  the OpenAPI for completeness and deliberately excluded from every skill in this repo — see
  `skills/_index.yml`.

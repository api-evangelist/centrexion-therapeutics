---
name: Harvest the media library safely
description: >-
  Enumerate media attachments from the Centrexion Therapeutics WordPress REST API while working
  around the X-WP-Total counting defect on this deployment, which over-reports the collection and
  traps a naive pager in empty pages.
api: openapi/centrexion-therapeutics-content-openapi.yml
operations: [listMedia, getMediaItem]
---

# Harvest the media library safely

## The defect you must code around

`GET /wp/v2/media` on centrexion.com **lies about how many items it will give you.** Measured
2026-08-09:

| `per_page` | `X-WP-Total` says | items actually returned |
|---|---|---|
| 1 | 43 | **0** |
| 5 | 43 | 2 |
| 10 | 43 | 7 |
| 100 | 43 | 26 |

Roughly 17 attachments are counted by the header but filtered out of the anonymous result set.
`X-WP-TotalPages` is computed from the inflated total, so it over-reports too.

Two concrete failure modes if you trust the headers:

1. **`per_page=1` returns zero items forever.** A one-at-a-time pager makes 43 requests and collects
   nothing.
2. **A `while page <= X-WP-TotalPages` loop runs past the real end** and burns requests on empty
   arrays.

## The rule

**Page to exhaustion. Stop on an empty array. Never derive your loop bound from `X-WP-Total` or
`X-WP-TotalPages`.**

## Steps

### 1. Enumerate with a large page size

Call `listMedia` (`GET /wp/v2/media`) with `per_page=100` — the maximum, and the size that yields the
most items here (26). Anything above 100 returns `400 rest_invalid_param`.

Trim the payload with `_fields=id,slug,title,source_url,mime_type,media_type,filesize,date`.
`media_details` in particular carries every generated size variant and is large.

### 2. Loop until empty, not until the count

```
page = 1
items = []
loop:
  r = GET /wp/v2/media?per_page=100&page=<page>&_fields=...
  if r is an empty array: stop
  items += r
  page += 1
  wait 10s          # robots.txt Crawl-delay
```

Also stop if the response is `400 rest_invalid_param` — WordPress returns that once `page` exceeds
the real page count.

### 3. Record the discrepancy rather than hiding it

When you report what you collected, state the retrieved count **and** that the API claimed 43. Do not
present 26 as "the media library" and do not present 43 as a count you verified. You retrieved 26;
the provider claims 43; the difference is unexplained from outside.

### 4. Fetch a single attachment when you need full metadata

Call `getMediaItem` (`GET /wp/v2/media/{id}`) for the complete object — `source_url`, `filesize`,
`mime_type`, `alt_text`, `caption`, and the size variants under `media_details`.

Known useful ids: **181** is the site logo (`Centrexion_LOGO_275px.png`), **189** is the favicon.

## What is in there

Across the 26 retrievable items observed 2026-08-09: 13 `image/jpeg`, 11 `image/png`, 1 `video/mp4`,
1 with an empty MIME type. Expect site chrome and stock imagery — this is a six-page marketing site,
not a media archive.

## Conventions

- **No conditional requests.** No ETag, no Last-Modified. You cannot revalidate cheaply; cache what
  you fetch on your side.
- **Responses are up to 10 minutes stale** (`cache-control: max-age=600`, WP Engine edge cache).
- **Filter server-side when you can** — `media_type=image` and `mime_type=image/png` both work and
  are cheaper than filtering client-side.
- **Files themselves are on the same origin.** `source_url` points at
  `https://centrexion.com/wp-content/uploads/...`; those are plain static fetches, not API calls, and
  are not subject to the pagination defect.

## Errors

| Status | `code` | Meaning |
|---|---|---|
| 404 | `rest_post_invalid_id` | No attachment with that id. |
| 400 | `rest_invalid_param` | `per_page` above 100, or `page` past the real end — treat the latter as your stop signal. |

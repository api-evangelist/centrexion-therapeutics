---
name: Discover and re-validate the API surface
description: >-
  Read the Centrexion Therapeutics route index, registered post types, taxonomies and statuses to
  establish what exists before calling anything, and to detect drift on a surface that carries no
  compatibility promise.
api: openapi/centrexion-therapeutics-content-openapi.yml
operations: [getApiIndex, listPostTypes, getPostType, listTaxonomies, getTaxonomy, listStatuses, listCategories, listTags]
---

# Discover and re-validate the API surface

## Why you re-discover instead of trusting a cached contract

Centrexion publishes no OpenAPI, no versioning policy and no deprecation policy. The contract in this
repo is an API Evangelist *derivation* of a route index that Centrexion can change at any time — a
plugin update, a theme change or a WordPress core upgrade will add or remove routes with no notice,
no `Sunset` header and no changelog. See `lifecycle/centrexion-therapeutics-lifecycle.yml`.

So: **discover first, then call.** This whole skill is four cheap GETs.

## Steps

### 1. Read the index

Call `getApiIndex` (`GET /`). You get:

- `name`, `description`, `home`, `timezone_string` — site identity.
- `namespaces` — 10 registered as of 2026-08-09: `oembed/1.0`, `wpe/cache-plugin/v1`,
  `wpe_sign_on_plugin/v1`, `aioseo/v1`, `sliderrevolution`, `google-site-kit/v1`, `wp/v2`,
  `wp-site-health/v1`, `wp-block-editor/v1`, `wp-abilities/v1`.
- `routes` — the full 320-entry route table.
- `authentication` — advertises WordPress application passwords.

Narrow it with `?namespace=wp/v2` when you only care about the content contract; the full index is
~270 KB.

**Assert `wp/v2` is present before proceeding.** If it is gone, stop — nothing else in this repo
applies.

### 2. Read the registered types

Call `listPostTypes` (`GET /wp/v2/types`). Thirteen anonymously as of 2026-08-09: `post`, `page`,
`attachment`, `nav_menu_item`, `wp_block`, `wp_template`, `wp_template_part`, `wp_global_styles`,
`wp_navigation`, `wp_font_family`, `wp_font_face`, `wpex_templates`, `wpex_card`.

**The important thing here is what is absent.** The custom post type holding the 14 staff biographies
is *not* in this list — it is not REST-registered, so it has no route and cannot be queried. If a
future run finds a `staff` type here, that is a real change worth acting on: the biographies would
have become structured data.

`getPostType` (`GET /wp/v2/types/{type}`) gives one registration record — `rest_base`, `taxonomies`,
`hierarchical`. An unregistered slug returns `404 rest_type_invalid`.

### 3. Read the taxonomies and their terms

Call `listTaxonomies` (`GET /wp/v2/taxonomies`). Five: `category`, `post_tag`, `nav_menu`,
`wp_pattern_category`, `post_series`. `getTaxonomy` fetches one.

Then check whether any are actually populated:

- `listCategories` (`GET /wp/v2/categories`) — only the default `Uncategorized` (id 1, count 1).
- `listTags` (`GET /wp/v2/tags`) — empty, `X-WP-Total: 0`.

Registered ≠ populated. All three content taxonomies are effectively unused, so do not build
taxonomy-driven navigation against this site.

### 4. Read the statuses

Call `listStatuses` (`GET /wp/v2/statuses`). Anonymously you get exactly one: `publish`. This is your
confirmation that you cannot see drafts, and why passing `status=draft` to a collection returns
`400 rest_invalid_param` rather than an empty list.

## Detecting drift between runs

Diff these four responses against the previous run and treat any of the following as a real change:

- A namespace added or removed from `getApiIndex`.
- A post type appearing in or vanishing from `listPostTypes` — **especially a staff/people type**.
- A taxonomy gaining terms where it previously had none.
- A route in the index that this repo's OpenAPI does not model.

Record what changed with the date you observed it. Do not silently update a cached contract.

## Where the surface stops

Not everything registered is readable. These returned `401` anonymously on 2026-08-09 and are **not**
modelled in the OpenAPI — do not probe them expecting data:

`/wp/v2/settings` (`rest_forbidden`), `/wp/v2/menus`, `/wp/v2/menu-locations`, `/wp/v2/icons`,
`/wp/v2/block-patterns/categories` (`rest_cannot_view`), `/wp/v2/themes`, `/wp/v2/plugins`,
`/wp/v2/block-types`, `/wp/v2/font-collections`, `/wp/v2/sidebars`, `/wp/v2/widget-types`,
`/wp/v2/pattern-directory/patterns`.

Note especially **`wp-abilities/v1`** — the WordPress Abilities API, an agent-facing capability
registry. Both `/wp-abilities/v1/abilities` and `/wp-abilities/v1/categories` return `401
rest_forbidden`. There is a registry, but it is closed. **Centrexion exposes no MCP server, no agent
card and no callable agent tools** — do not claim otherwise on the strength of the namespace merely
being registered.

Full gated list with per-route error codes:
`authentication/centrexion-therapeutics-authentication.yml` under `gated_surface`.

## Pace

`robots.txt` asks for `Crawl-delay: 10`. These are four requests — spread them out and cache the
result; the answers change on the order of months, not minutes.

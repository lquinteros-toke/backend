---
icon: bag-shopping
---

# Products

Calls go from Flutter to Gateway, which proxies to BudHub and returns the data.

## Overview

Cannabis products use a polymorphic schema in BudHub — each product is one row in `product_base` joined to one row in `product_cannabis`. Strain-specific data (cannabinoids, terpenes, feelings, etc.) lives in `strain_variant_datasource`, with fallbacks to static fields on `strain_variant`.

Two endpoints serve two distinct shapes: a paginated list for catalog browsing and a single-product fetch for detail pages.

The list endpoint uses **direct SQL** in BudHub for filtering, sorting and pagination — all heavy work runs in Postgres, returning only the page the user sees. Composition filters (cannabinoid percentage range, terpene mg/g range, feeling/use/profile intensity + population ranges) are pre-resolved in BudHub before the main query runs.

The detail endpoint uses **Xano addons** for related data (images, business) since addon resolution outperforms function calls for single-row fetches.

Both endpoints are accessed via the BudHub proxy on Gateway. Flutter sends a POST to Gateway with the target BudHub path and any filters. Gateway forwards the request to BudHub as a GET, then returns the response.

## Flow Overview

```
Flutter → Gateway POST /budhub (gateway_token)
           │
           ├─ Validates gateway_token
           ├─ Forwards to BudHub GET /cannabis (or /cannabis/single/{id})
           ├─ Passes query_params as GET params
           │
           ▼
         Returns paginated product list or single product detail
```

## Endpoints

### POST /budhub — Cannabis Products List

Generic proxy endpoint that forwards the request to BudHub's cannabis list endpoint. Returns a paginated, filtered, sorted list of listed cannabis products. Filters out orphan products (those in `product_base` without a matching `product_cannabis` row).

Called on: Gateway directly from Flutter\
Auth: `<gateway_token>`

**Headers:**

| Header        | Value                    | Required |
| ------------- | ------------------------ | -------- |
| Authorization | Bearer `<gateway_token>` | Yes      |
| Content-Type  | application/json         | Yes      |
| X-Data-Source | test or live             | Yes      |
| X-Branch      | dev or v1                | Yes      |

**Body:**

| Field          | Type   | Required | Notes                                                                                |
| -------------- | ------ | -------- | ------------------------------------------------------------------------------------ |
| `url`          | string | Yes      | Full BudHub endpoint URL:  `https://xads-j1s5-11eg.f2.xano.io/api:POeNg3uV/cannabis` |
| `branch`       | string | Yes      | dev or v1                                                                            |
| `query_params` | object | Yes      | Filter, sort, and pagination parameters (see below)                                  |

**`query_params` for cannabis list:**

| Param                | Type      | Required | Notes                                                                                   |
| -------------------- | --------- | -------- | --------------------------------------------------------------------------------------- |
| `page`               | int       | No       | Default `1`                                                                             |
| `per_page`           | int       | No       | Default `20`                                                                            |
| `sort_by`            | enum      | No       | `created_at` (default), `name`, `review_average`, `review_count`                        |
| `sort_order`         | enum      | No       | `asc`, `desc` (default)                                                                 |
| `cannabis_type_id`   | int       | No       | REC / MMJ / CBD                                                                         |
| `strain_type_id`     | int       | No       | Indica / Hybrid / Sativa                                                                |
| `strain_id`          | int       | No       |                                                                                         |
| `business_id`        | int       | No       | Brand filter                                                                            |
| `category_id`        | int       | No       |                                                                                         |
| `subcategory_id`     | int       | No       |                                                                                         |
| `irradiation`        | bool      | No       |                                                                                         |
| `rating_range`       | object    | No       | `{min, max}` decimals — review\_average bounds                                          |
| `variant_size_range` | object    | No       | `{min, max}` decimals — variant size bounds                                             |
| `price_range`        | object    | No       | `{min, max}` decimals — variant rrp bounds                                              |
| `cannabinoids`       | object\[] | No       | `[{cannabinoid_id, percentage_min, percentage_max}, ...]`                               |
| `terpenes`           | object\[] | No       | `[{terpene_id, mg_per_g_min, mg_per_g_max}, ...]`                                       |
| `feelings`           | object\[] | No       | `[{feeling_id, intensity_min?, intensity_max?, population_min?, population_max?}, ...]` |
| `uses`               | object\[] | No       | Same shape as feelings (`use_id` instead of `feeling_id`)                               |
| `profiles`           | object\[] | No       | Same shape as feelings (`profile_id` instead of `feeling_id`)                           |

**Filter semantics:**

* All filters combine with AND.
* Composition arrays AND across items: passing two `cannabinoids` entries means products must match both.
* Range bounds are inclusive (`BETWEEN min AND max`).
* `variant_size_range` and `price_range` match if the product has _any_ variant in range.
* Omit a filter entirely (or send `null`) to skip it. Sending `0` is treated as a real filter value.

**Example request:**

```json
POST https://xads-j1s5-11eg.f2.xano.io/api:oGtZdRL8/budhub

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: v1

Body:
{
  "url": "https://xads-j1s5-11eg.f2.xano.io/api:POeNg3uV/cannabis",
  "branch": "dev",
  "query_params": {
  "page": 1,
  "per_page": 20,
  "sort_by": "review_average",
  "sort_order": "desc",
  "cannabis_type_id": 3,
  "strain_type_id": 1,
  "strain_id": 3843,
  "business_id": 44,
  "category_id": 2,
  "subcategory_id": 5,
  "irradiation": false,
  "rating_range": {
    "min": 3.5,
    "max": 5.0
  },
  "variant_size_range": {
    "min": 1,
    "max": 10
  },
  "price_range": {
    "min": 10,
    "max": 100
  },
  "cannabinoids": [
    {
      "cannabinoid_id": 6,
      "percentage_min": 20,
      "percentage_max": 100
    },
    {
      "cannabinoid_id": 21,
      "percentage_min": 0,
      "percentage_max": 30
    }
  ],
  "terpenes": [
    {
      "terpene_id": 6,
      "mg_per_g_min": 5,
      "mg_per_g_max": 50
    }
  ],
  "feelings": [
    {
      "feeling_id": 1,
      "intensity_min": 3.0,
      "intensity_max": 5.0,
      "population_min": 50,
      "population_max": 100
    }
  ],
  "uses": [
    {
      "use_id": 1,
      "intensity_min": 2.0,
      "intensity_max": 5.0,
      "population_min": 30,
      "population_max": 100
    }
  ],
  "profiles": [
    {
      "profile_id": 42,
      "intensity_min": 1.0,
      "intensity_max": 5.0,
      "population_min": 10,
      "population_max": 100
    }
  ]
}
}
```

**Response:**

Returns a paginated array of cannabis products with full hydration.

```json
{
  "items": [
    {
      "base": {
        "id": 3,
        "name": "...",
        "sku": "...",
        "review_average": 4.5,
        "review_count": 12,
        "...": "..."
      },
      "cannabis": {
        "id": 1,
        "cannabis_type_id": 3,
        "strain_type_id": 1,
        "strain_variant_id": 8408,
        "strain_id": 3843,
        "irradiation": false,
        "product_origin": "manual"
      },
      "strain_data": {
        "cannabinoids": [
          {
            "cannabinoid_id": 6,
            "percentage": 79.6,
            "short_name": "CBD",
            "name": "Cannabidiol",
            "image_url": "",
            "frequency_type": "Common"
          }
        ],
        "terpenes": [
          {
            "terpene_id": 6,
            "mg_per_g": 29,
            "short_name": "β-Pinene",
            "name": "",
            "image_url": "",
            "frequency_type": "Common"
          }
        ],
        "profiles": [
          { "profile_id": 42, "intensity": 1.41, "population": 11.73, "name": "Mossy" }
        ],
        "feelings": [
          { "feeling_id": 1, "intensity": 4.95, "population": 70.73, "name": "" }
        ],
        "uses": [
          { "use_id": 1, "intensity": 3.44, "population": 45.9, "name": "Alzheimer's" }
        ],
        "side_effects": [
          { "side_effect_id": 27, "intensity": 3.28, "population": 28 }
        ],
        "activities": [
          { "activity_id": 1, "intensity": 2, "population": 0, "name": "Biking" }
        ]
      },
      "variants": [
        { "size": 3, "unit_id": 3, "rrp": 23 }
      ]
    }
  ],
  "itemsTotal": 1,
  "pageTotal": 1,
  "curPage": 1,
  "nextPage": null,
  "prevPage": null,
  "perPage": 20
}
```

**Errors:**

| Status | Message                              | When                                                            |
| ------ | ------------------------------------ | --------------------------------------------------------------- |
| 401    | Unauthorized                         | Invalid or missing gateway\_token                               |
| 500    | Cannabis product type not configured | The Cannabis row is missing from `product_type` table in BudHub |

**Notes:**

* Only products with `listing_status == "Listed"` are returned.
* Products without a corresponding `product_cannabis` row are excluded.
* `pageTotal` is integer-rounded (ceiling division of `itemsTotal / per_page`).
* `nextPage` and `prevPage` are `null` when navigation is unavailable (out-of-bounds pages, no results, or page 1).
* When composition filters are active and no strain\_variant matches, the resolver short-circuits and the main query is skipped — returns empty `items` with correct pagination metadata.
* Composition filtering uses pre-built strain\_variant\_id allowlists from `strain_variant_datasource`. For products whose strain\_variant has no matching datasource row for the requested element, the filter excludes them (no fallback to static fields for filtering — fallback only applies to display).
* Variants in the list response are slim (`{size, unit_id, rrp}` only). The detail endpoint returns full variant objects with images and `cannabis_product_variant` data.

***

### POST /budhub — Single Cannabis Product

Generic proxy endpoint that forwards the request to BudHub's single cannabis product endpoint. Returns full detail for one product, including business info and per-variant images.

Called on: Gateway directly from Flutter\
Auth: `gateway_token`

**Headers:**

| Header        | Value                    | Required |
| ------------- | ------------------------ | -------- |
| Authorization | Bearer `<gateway_token>` | Yes      |
| Content-Type  | application/json         | Yes      |
| X-Data-Source | test or live             | Yes      |
| X-Branch      | dev or v1                | Yes      |

**Body:**

| Field          | Type   | Required | Notes                                                                                                                  |
| -------------- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------- |
| `url`          | string | Yes      | BudHub single cannabis endpoint URL https://xads-j1s5-11eg.f2.xano.io/api:POeNg3uV/cannabis/single/{product\_base\_id} |
| `branch`       | string | Yes      | dev or v1                                                                                                              |
| `query_params` | object | No       | None required for this endpoint                                                                                        |

**Path parameter (in `url`):**

| Param             | Type | Required | Notes                                |
| ----------------- | ---- | -------- | ------------------------------------ |
| `product_base_id` | int  | Yes      | Substituted into the BudHub URL path |

**Example request:**

```json
POST https://xads-j1s5-11eg.f2.xano.io/api:oGtZdRL8/budhub

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: v1

Body:
{
  "url": "https://xads-j1s5-11eg.f2.xano.io/api:POeNg3uV/cannabis/single/3",
  "branch": "dev",
  "query_params": {}
}
```

**Response:**

Returns a single cannabis product with hydrated business info, per-variant images, and full strain data.

```json
{
  "base": {
    "id": 3,
    "name": "...",
    "sku": "...",
    "review_average": 4.5,
    "review_count": 12,
    "product_image_id": [
      { "id": 17, "file_link": "...", "primary": true }
    ],
    "_business": {
      "name": "GreenCo",
      "about": "...",
      "logo_horizontal_url": "..."
    },
    "...": "..."
  },
  "cannabis": {
    "id": 1,
    "cannabis_type_id": 3,
    "strain_type_id": 1,
    "strain_variant_id": 8408,
    "strain_id": 3843,
    "irradiation": false,
    "product_origin": "manual"
  },
  "strain_data": {
    "cannabinoids": [
      {
        "cannabinoid_id": 6,
        "percentage": 79.6,
        "short_name": "CBD",
        "name": "Cannabidiol",
        "image_url": "",
        "frequency_type": "Common"
      }
    ],
    "terpenes": [...],
    "profiles": [...],
    "feelings": [...],
    "uses": [...],
    "side_effects": [...],
    "activities": [...]
  },
  "variants": [
    {
      "variant_base": {
        "id": 7,
        "size": 3,
        "unit_id": 3,
        "rrp": 23,
        "product_image_id": [
          { "id": 22, "file_link": "...", "primary": true }
        ]
      },
      "cannabis_variant": {
        "id": 5,
        "strain_variant_id": 8408
      }
    }
  ]
}
```

**Errors:**

| Status | Message                           | When                                                                 |
| ------ | --------------------------------- | -------------------------------------------------------------------- |
| 401    | Unauthorized                      | Invalid or missing gateway\_token                                    |
| 404    | Product not found                 | No `product_base` row with that ID                                   |
| 404    | Product is not a cannabis product | Product exists but its `product_type_id` doesn't resolve to Cannabis |

**Notes:**

* Returns hydrated business info via the `_business` addon (list endpoint does not).
* Returns per-variant images via the `product_image` addon on `product_variant_base` (list endpoint returns slim variants without images).
* Returns `cannabis_product_variant` per variant (list endpoint omits this).
* Strain enrichment follows the same logic as the list endpoint: prefer datasource row for the element, fall back to `strain_variant.<element>_temp` for feelings/uses/profiles/side\_effects/activities (or `strain_variant.cannabinoids` / `strain_variant.terpenes` for those two element names), default to empty array if neither exists.

## Datasource awareness

Both endpoints work in `live` and `test` datasources. The list endpoint uses `$env.$datasource` to interpolate the right table names into its SQL (`x2_234` live vs `x2_test_234` test for `product_base`, etc.). The detail endpoint uses standard `db.get`/`db.query` which Xano routes automatically based on the `X-Data-Source` header.

## Why two endpoints with different shapes

| Concern                      | List                                                                                                         | Detail                                                                          |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| Use case                     | Catalog browse, Flutter list scroll                                                                          | Detail page, tap into product                                                   |
| Filtering                    | 16 filter dimensions                                                                                         | None (single-row fetch by ID)                                                   |
| Pagination                   | Yes                                                                                                          | N/A                                                                             |
| Strategy                     | Direct SQL → hydrator function                                                                               | `db.get` with addons → inline strain enrichment                                 |
| Image data on `product_base` | Not included                                                                                                 | `product_image_id[]` array of resolved image rows                               |
| Business data                | Not included                                                                                                 | `_business` object                                                              |
| Variant data                 | Slim (`{size, unit_id, rrp}`)                                                                                | Full (`variant_base` + images + `cannabis_variant`)                             |
| Why this design              | At list scale, addons can't batch across N rows efficiently — direct SQL with single hydrator call is faster | At detail scale, Xano's addon resolver beats function-call overhead for one row |

The shapes intentionally diverge. The list endpoint optimizes for "show me 20 cards quickly"; the detail endpoint optimizes for "show me everything about one product."

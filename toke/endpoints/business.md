---
icon: briefcase-blank
---

# Business

How businesses from BudHub are synced, displayed, and interacted with in the Toke consumer app. Businesses are the core entity shown on the map and in listing views.

## Overview

Toke does not store business data in BudHub directly. Instead, it maintains a local  `business_snapshot`  table that mirrors BudHub business data in a consumer-optimized format. This snapshot is kept in sync through two mechanisms: a daily background task and real-time event pushes when businesses are created, edited, or deactivated in BudHub.

The map feature relies on two endpoints: a lightweight  nearby  endpoint for rendering map pins and tile cards (\~50ms), and a  detail  endpoint that fetches live data from BudHub for the expanded business view (\~200-400ms).



## How data flows between apps

When a Toke user opens the map, Flutter calls Toke's nearby endpoint directly using the  `toke_token` . This reads from the local snapshot table. No Gateway or BudHub calls involved.

When the user taps a business tile for full details, Flutter calls the detail endpoint which reaches BudHub through the Gateway for data, then merges it with local reviews and ratings.

<table><thead><tr><th valign="top">Caller</th><th valign="top">Destination</th><th valign="top">Path</th><th valign="top">Auth Method</th></tr></thead><tbody><tr><td valign="top">Flutter (Toke)</td><td valign="top">Toke DB (nearby)</td><td valign="top">Direct</td><td valign="top">toke_token (Xano auth)</td></tr><tr><td valign="top">Flutter (Toke)</td><td valign="top">BudHub DB (detail)</td><td valign="top">Gateway proxy</td><td valign="top">toke_token → Gateway → BudHub</td></tr><tr><td valign="top">BudHub (push)</td><td valign="top">Toke DB (sync)</td><td valign="top">Gateway proxy</td><td valign="top">X-Service-Key → Gateway → app_key</td></tr><tr><td valign="top">Toke (daily task)</td><td valign="top">BudHub DB (export)</td><td valign="top">Gateway proxy</td><td valign="top">app_key → Gateway → X-Service-Key</td></tr></tbody></table>

## Endpoints

### GET /nearby

Returns businesses within a radius of the given coordinates as a GeoJSON FeatureCollection for Mapbox. Supports filtering by cannabis type, delivery type, and business type.

**Auth:** Xano built-in (`toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

| Input                  | Type    | Required | Description                                                  |
| ---------------------- | ------- | -------- | ------------------------------------------------------------ |
| `latitude`             | Decimal | Yes      | User's current or alternate latitude                         |
| `longitude`            | Decimal | Yes      | User's current or alternate longitude                        |
| `radius_km`            | Decimal | No       | Search radius in kilometers                                  |
| `cannabis_type_filter` | Text    | No       | Filter by short\_name: `CBD`, `REC`, `MMJ`                   |
| `delivery_type_filter` | Text    | No       | Filter by: `Delivery`, `Collection`, `Collection & Delivery` |
| `business_type_filter` | Text    | No       | Filter by: `Brand`, `Dispensary`                             |

**Example request:**

```json
{
  "latitude": 51.505316,
  "longitude": -0.09936249999999999,
  "radius_km": 100,
  "cannabis_type_filter": "1",
  "delivery_type_filter": "Delivery",
  "business_type_filter": "Dispensary"
}
```

**Response**

GeoJSON FeatureCollection:

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [-0.09936, 51.50532]
      },
      "properties": {
        "id": 18,
        "bh_business_id": 48,
        "name": "Dispensary 9",
        "business_type": "Dispensary",
        "thumbnail_url": "//cdn.bubble.io/.../logo.jpg",
        "cannabis_type": [...],
        "default_cannabis_type": {...},
        "avg_rating": 4.2,
        "review_count": 15,
        "delivery_type": "Collection",
        "opening_times": [...],
        "address_full": "71-79 Southwark St, London SE1 0JA, UK",
        "distance_km": 0.8,
        "retailer_type": "Cannabis",
        "max_members_count": 100,
        "members_count": 60
      }
    }
  ]
}
```

**Errors:**

| Status | Message       | When                           |
| ------ | ------------- | ------------------------------ |
| `401`  | Access Denied | Invalid or missing toke\_token |

Notes:

* GeoJSON coordinates are `[longitude, latitude]` (Mapbox standard)
* `distance_km` is calculated via `util.geo_distance` and rounded to 1 decimal
* Only `is_active == true` businesses are returned

***

### GET /business/{bh\_business\_id}

Returns full live business data from BudHub plus reviews and ratings from Toke. Single call for the Flutter detail view.

**Auth:** Xano built-in (`toke_token`)

**Headers**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Path parameters:**

| Input            | Type    | Required | Description             |
| ---------------- | ------- | -------- | ----------------------- |
| `bh_business_id` | Integer | Yes      | The BudHub business ID  |

**Example request:**

```json
GET https://xads-j1s5-11eg.f2.xano.io/api:rOS4jyzC/business/17
```

#### Flow

1. Calls Gateway `GET /service/business-export?business_id={id}` to get live BudHub data
2. Transforms social media via `transform_social_media`
3. Transforms opening times via `transform_opening_times`
4. Reads `avg_rating` and `review_count` from local `business_snapshot`
5. Queries last 5 published, public reviews from local `review` table
6. Merges everything into a single response

#### Response

```json
{
  "bh_business_id": 17,
  "name": "3CBD Paradise (Dispensary)",
  "slug": "string",
  "business_type": "Dispensary",
  "cannabis_type": [{...}],
  "default_cannabis_type": {...},
  "thumbnail_url": "//cdn.bubble.io/.../logo.jpg",
  "main_image_url": "//cdn.bubble.io/.../cover.jpg",
  "logo_horizontal_url": "//cdn.bubble.io/.../logo_h.jpg",
  "overview": "Business description...",
  "address_full": "Gran Vía, 3, Madrid, Spain",
  "address_2": "",
  "country_name": "Spain",
  "latitude": 40.4191902,
  "longitude": -3.6985002,
  "delivery_type": "Delivery",
  "private": false,
  "opening_times": [
    {"day": "Monday", "open": true, "slots": [{"from": "10:00", "to": "12:00"}]},
    ...
  ],
  "website": "example.com",
  "email": "contact@example.com",
  "phone": "+34666666663",
  "social_media": [
    {"platform": "Instagram", "url": "cbdparadiseIG3"},
    ...
  ],
  "operation_countries": [{"name": "Spain", "ISO_alpha_2": "ES"}],
  "status": "Active",
  "retailer_type": "Cannabis",
  "max_members_count": 100,
  "members_count": 60,
  "products_base": [1,2]
  "avg_rating": 2,
  "review_count": 3,
  "reviews": {
    "itemsReceived": 3,
    "curPage": 1,
    "items": [
      {
        "id": 3,
        "overall_rating": 1,
        "title": "Review title",
        "body": "Review body",
        "reviewer_type": "Non-verified",
        "created_at": 1773535859385,
        ...
      }
    ]
  }
}
```

**Errors:**

| Status | Message            | When                                            |
| ------ | ------------------ | ----------------------------------------------- |
| `401`  | Access Denied      | Invalid or missing toke\_token                  |
| `404`  | Business not found | Business doesn’t exist or export returned empty |

***

### POST /business-sync

Receives a full business object from the Gateway and upserts it into `business_snapshot`. Used for real-time event push when a business is created, edited, or deactivated in BudHub.

**Auth:** `app_key` verification against `TOKE_APP_KEY` env var

**Headers**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

| Input      | Type | Required | Description                                          |
| ---------- | ---- | -------- | ---------------------------------------------------- |
| `app_key`  | Text | Yes      | The app key identifying the calling app              |
| `business` | JSON | Yes      | The full resolved business object from BudHub export |

#### Flow

1. Validates `app_key` matches `TOKE_APP_KEY`
2. Builds `address_full` from address parts
3. Transforms social media (with try/catch fallback)
4. Transforms opening times (with try/catch fallback)
5. Cleans `address_2`
6. Determines `is_active` from `business.status == "Active"`
7. Upserts into `business_snapshot` by `bh_business_id`

#### Response

```json
{"success": true}
```

**Errors:**

| Status | Message      | When             |
| ------ | ------------ | ---------------- |
| `401`  | Unauthorized | Invalid app\_key |

***

## Functions

### transform\_social\_media

Cleans and deduplicates the raw `social_business_id` array from BudHub into a flat array of `{platform, url}` objects.

**Input:**

| Input        | Type | Description                                       |
| ------------ | ---- | ------------------------------------------------- |
| `raw_social` | JSON | The `social_business_id` array from BudHub export |

**Logic:**

1. Loops through the raw array
2. Skips null entries
3. Extracts `platform` from `_social_media.name` and `url` from `url`
4. Deduplicates by `url` only the first occurrence is kept

**Output:**

```json
[
  {"platform": "Instagram", "url": "cbdparadiseIG3"},
  {"platform": "Facebook", "url": "cbdparadiseFB3"},
  {"platform": "X", "url": "cbdparadiseX3"}
]
```

**Called by:** `sync_business_snapshot` task, `POST /service/business-sync`, `GET /businesses/{bh_business_id}`

***

### transform\_opening\_times

Groups BudHub's flat opening\_times array into a per-day structure with time slots. Skips closed days and empty slots.

**Input:**

| Input       | Type | Description                                  |
| ----------- | ---- | -------------------------------------------- |
| `raw_times` | JSON | The `opening_times` array from BudHub export |

**Logic:**

1. Loops through raw times, skipping entries where `open == false` or `from` is empty
2. Groups slots by day name (from `_week_days.day`)
3. Multiple time slots per day are collected into a `slots` array
4. Returns all 7 days in order (Monday–Sunday), with `open: false` and empty `slots` for days with no entries

**Output:**

```json
[
  {
    "day": "Monday",
    "open": true,
    "slots": [
      {"from": "10:00", "to": "12:00"},
      {"from": "12:30", "to": "14:00"},
      {"from": "16:00", "to": "18:00"}
    ]
  },
  {
    "day": "Tuesday",
    "open": false,
    "slots": []
  }
]
```

**Note:** Entries where `from` contains Unix timestamps (e.g. `"1770886800000"`) instead of `"HH:MM"` strings are included as-is — Flutter must handle validation. This is a BudHub data quality issue.

**Called by:** `sync_business_snapshot` task, `POST /service/business-sync`, `GET /businesses/{bh_business_id}`

***

### update\_business\_rating

Recalculates `avg_rating` and `review_count` on the `business_snapshot` table based on published, public reviews.

**Input:**

| Input            | Type    | Description                                       |
| ---------------- | ------- | ------------------------------------------------- |
| `bh_business_id` | Integer | The BudHub business ID to recalculate ratings for |

**Logic:**

1. Counts all reviews where `bh_reviewable_id = bh_business_id`, `reviewable_type = "business"`, `status = "Published"`, `is_public = true`
2. If count > 0, sums all `overall_rating` values and divides by count, rounded to 1 decimal
3. Updates `avg_rating` and `review_count` on the matching `business_snapshot` record
4. If no reviews exist, sets both to 0

**Output:**

```json
{
  "avg_rating": 3.7,
  "review_count": 12
}
```

**Called by:** Any review create, edit, or delete endpoint (add `function.call update_business_rating` at the end)

***

## Ecosystem Flow: Business Lifecycle

### Creating a New Business

{% stepper %}
{% step %}
Admin creates business in Bubble (BudHub frontend).
{% endstep %}

{% step %}
Bubble calls BudHub POST /business/create

* Creates business record in BudHub DB
* Status defaults to "Pending"
* Calls geocode\_address → stores lat/lng
{% endstep %}

{% step %}
BudHub post-process pushes to Gateway\
POST /service/business-sync\
Body: { business\_id: 117, action: "created", branch: "dev" }\
Header: X-Service-Key
{% endstep %}

{% step %}
Gateway receives the push

* Validates service key (gateway\_service\_middleware)
* Calls BudHub GET /business/export?business\_id=117
* Gets full resolved business object
{% endstep %}

{% step %}
Gateway forwards to Toke\
POST /service/business-sync\
Body: { app\_key, business: {resolved object} }
{% endstep %}

{% step %}
Toke receives and processes:

* Transforms social media
* Transforms opening times
* status = "Pending" → is\_active = false
* Upserts into business\_snapshot
{% endstep %}

{% step %}
Result: Business exists in snapshot but is NOT visible on the map (is\_active = false)
{% endstep %}
{% endstepper %}

***

### Approving a Business (Pending → Active)

{% stepper %}
{% step %}
Admin approves business in Bubble.
{% endstep %}

{% step %}
Bubble calls BudHub PATCH /business/{id}/status\
Body: { status: "Active" }
{% endstep %}

{% step %}
BudHub pushes to Gateway\
{ business\_id: 117, action: "updated" }
{% endstep %}

{% step %}
Gateway fetches → forwards to Toke.
{% endstep %}

{% step %}
Toke upserts:

* status = "Active" → is\_active = true
{% endstep %}

{% step %}
Result: Business is NOW visible on the map.
{% endstep %}
{% endstepper %}

***

### Editing a Business

{% stepper %}
{% step %}
Admin edits business data in Bubble (name, address, hours, social media, etc.).
{% endstep %}

{% step %}
Bubble calls BudHub PATCH /business/{id}

* Updates business record
* If address changed → re-runs geocode\_address
{% endstep %}

{% step %}
BudHub pushes to Gateway\
{ business\_id: 117, action: "updated" }
{% endstep %}

{% step %}
Gateway fetches fresh data → forwards to Toke.
{% endstep %}

{% step %}
Toke upserts with new data:

* All fields updated
* avg\_rating and review\_count preserved (not in upsert)
{% endstep %}

{% step %}
Result: Map and detail views show updated data.
{% endstep %}
{% endstepper %}

***

### Deactivating a Business

{% stepper %}
{% step %}
Admin deactivates business in Bubble.
{% endstep %}

{% step %}
Bubble calls BudHub PATCH /business/{id}/status\
Body: { status: "Archived" }
{% endstep %}

{% step %}
BudHub pushes to Gateway\
{ business\_id: 117, action: "deactivated" }
{% endstep %}

{% step %}
Gateway fetches → forwards to Toke.
{% endstep %}

{% step %}
Toke upserts:

* status = "Archived" → is\_active = false
{% endstep %}

{% step %}
Result: Business disappears from the map. Record still exists (preserves reviews). Can be reactivated later.
{% endstep %}
{% endstepper %}

***

## Background Tasks

### sync\_business\_snapshot

Pulls all businesses from BudHub via Gateway service endpoint, transforms data, and upserts into `business_snapshot`. Sets `is_active` based on BudHub status. Marks businesses deleted from BudHub as inactive.

**Schedule:** Daily at 04:00 UTC (86400 second interval)

**Flow:**

{% stepper %}
{% step %}
Calls Gateway `GET /service/business-export` with `app_key` and `branch`.
{% endstep %}

{% step %}
Validates response status == 200.
{% endstep %}

{% step %}
Loops through each business:

* Builds `address_full` from address parts
* Calls `transform_social_media` (with try/catch fallback to `[]`)
* Calls `transform_opening_times` (with try/catch fallback to `[]`)
* Cleans `address_2` (converts `"null"` string to `""`)
* Safely extracts `_cannabis_type` (may not exist)
* Determines `is_active` from `status == "Active"`
* Upserts into `business_snapshot` by `bh_business_id`
{% endstep %}

{% step %}
After all upserts, queries all active snapshots and deactivates any whose `bh_business_id` wasn't in the export (sets `bh_status = "Deleted"`).
{% endstep %}
{% endstepper %}

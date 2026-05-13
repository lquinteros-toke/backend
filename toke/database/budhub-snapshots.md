---
icon: camera-polaroid
---

# BudHub Snapshots

Consumer-optimized snapshot of BudHub business data. Used for map view, business listings, and detail cards in the Toke app.

***

## business\_snapshot

Consumer-optimized snapshot of BudHub business data. Used for map view, business listings, and detail cards in the Toke app.

| Field                   | Type      | Description                                                                                    |
| ----------------------- | --------- | ---------------------------------------------------------------------------------------------- |
| `id`                    | Integer   | Primary key                                                                                    |
| `created_at`            | Timestamp | Record creation time                                                                           |
| `bh_business_id`        | Integer   | BudHub business ID. Cross-reference key for syncing. **Unique.**                               |
| `name`                  | Text      | Business display name                                                                          |
| `slug`                  | Text      | URL-friendly identifier for deep linking                                                       |
| `business_type`         | Enum      | `Brand` or `Dispensary`                                                                        |
| `cannabis_type`         | JSON      | Array of cannabis type objects: `[{name, short_name, font_color, background_color, map_icon}]` |
| `default_cannabis_type` | JSON      | Single cannabis type object used as primary display                                            |
| `thumbnail_url`         | Text      | Small image URL (logo) for map tiles and list cards                                            |
| `main_image_url`        | Text      | Cover image URL for expanded detail view                                                       |
| `logo_horizontal_url`   | Text      | Horizontal logo for wider layouts                                                              |
| `overview`              | Text      | Short business description                                                                     |
| `address_full`          | Text      | Concatenated display address                                                                   |
| `address_2`             | Text      | Second address line                                                                            |
| `country_name`          | Text      | Resolved country name                                                                          |
| `latitude`              | Decimal   | Geographic latitude for radius queries                                                         |
| `longitude`             | Decimal   | Geographic longitude for radius queries                                                        |
| `delivery_type`         | Text      | `Delivery`, `Collection`, or `Collection & Delivery`                                           |
| `opening_times`         | JSON      | Grouped by day: `[{day, open, slots: [{from, to}]}]`                                           |
| `website`               | Text      | Business website URL                                                                           |
| `email`                 | Text      | Contact email                                                                                  |
| `phone`                 | Text      | Contact phone number                                                                           |
| `social_media`          | JSON      | Deduplicated array: `[{platform, url}]`                                                        |
| `operation_countries`   | JSON      | Array: `[{name, ISO_alpha_2}]`                                                                 |
| `avg_rating`            | Decimal   | Aggregated average star rating from Toke reviews                                               |
| `review_count`          | Integer   | Total number of published public reviews                                                       |
| `is_active`             | Boolean   | Whether visible to consumers. Driven by BudHub `status`.                                       |
| `bh_status`             | Text      | BudHub business status: `Active`, `Pending`, `Rejected`, `Archived`                            |
| `last_synced_at`        | Timestamp | When this record was last refreshed from BudHub                                                |
| `retailer_type`         | Enum      | `Cannabis`, `Non-Cannabis`, `Lounge/Bar`, `Club`                                               |
| `max_members_count`     | Integer   | Maximum number of members allowed to join                                                      |
| `members_count`         | Integer   | Current number of members                                                                      |

### Computed Fields (not synced from BudHub)

| Field          | Source                                                    | Updated when                            |
| -------------- | --------------------------------------------------------- | --------------------------------------- |
| `avg_rating`   | Toke `review` table via `update_business_rating` function | A review is created, edited, or deleted |
| `review_count` | Toke `review` table via `update_business_rating` function | A review is created or deleted          |

### Sync Strategy

**Daily Sync (safety net):**\
Background task `sync_business_snapshot` runs daily at 4:00 AM UTC. Calls Gateway → BudHub `GET /business/export`. Upserts all businesses by `bh_business_id`. Sets `is_active` based on BudHub `status` field.

**Event Push (real-time updates):**\
When a business is created, edited, or deactivated in BudHub, it pushes the `business_id` and `action` to the Gateway. Gateway fetches the resolved business from BudHub's export endpoint, then forwards it to Toke's `POST /service/business-sync`. Toke upserts the record with transformed data.

#### Deactivation Behavior

`is_active` is always driven by BudHub `status`:

| BudHub status | `is_active` |
| ------------- | ----------- |
| `Active`      | `true`      |
| `Pending`     | `false`     |
| `Rejected`    | `false`     |
| `Archived`    | `false`     |
| `""` (empty)  | `false`     |
| `null`        | `false`     |

Records are never deleted — only marked inactive. This preserves review references.

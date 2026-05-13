---
icon: file-magnifying-glass
---

# Overview

Toke is the consumer-facing application of the ecosystem. It serves as the mobile app (Flutter) where end users browse cannabis products, place orders, write reviews, and manage their account.

***

## What Toke stores

Toke DB holds **consumer-specific data only.** Everything that belongs to the user and their experience within the app. Products, strains, dispensaries, and business data live in BudHub and are accessed through the Gateway.

| Category         | Tables                                                                                                              | Description                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Users            | `user`, `user_health`, `user_medication`, `user_address`                                                            | Consumer accounts, onboarding data, health profiles, and delivery addresses             |
| Reviews          | `review`, `review_feeling`, `review_use`, `review_profile`, `review_side_effect`, `review_comment`, `review_report` | Product and dispensary reviews with properties, comments, votes, and moderation reports |
| Commerce         | `basket`, `basket_item`, `user_subscription`, `user_credit`                                                         | Shopping baskets, subscription management, and credit transactions                      |
| Lookups          | `currency`, `habits_frequency`, `report_reason`                                                                     | Local reference tables                                                                  |
| Budhub Snapshots | `business_snapshot`                                                                                                 | Simplified information pulled from BudHub DB and stored in Toke                         |

***

## Who accesses Toke

| Caller              | How                              | Purpose                                                          |
| ------------------- | -------------------------------- | ---------------------------------------------------------------- |
| **Flutter app**     | Directly with `toke_token`       | All consumer actions: browse, review, purchase, manage profile   |
| **BudHub (Bubble)** | Via Gateway with `gateway_token` | Admin moderation: manage users, moderate reviews, adjust credits |

***

## Cross-database references

Toke tables reference BudHub data using plain integer fields prefixed with `bh_`. These are not foreign keys, they are stored IDs that the frontend resolves through the Gateway.

<table><thead><tr><th width="272.2962646484375">Toke field</th><th width="242.9630126953125">On table</th><th>References in BudHub</th></tr></thead><tbody><tr><td><code>bh_reviewable_id</code></td><td><code>review</code></td><td><code>product</code> or <code>business</code> (depends on <code>reviewable_type</code>)</td></tr><tr><td><code>bh_country_id</code></td><td><code>review</code></td><td><code>country</code></td></tr><tr><td><code>bh_feeling_id</code></td><td><code>review_feeling</code></td><td><code>feeling</code></td></tr><tr><td><code>bh_use_id</code></td><td><code>review_use</code></td><td><code>use</code></td></tr><tr><td><code>bh_profile_id</code></td><td><code>review_profile</code></td><td><code>profile</code></td></tr><tr><td><code>bh_side_effect_id</code></td><td><code>review_side_effect</code></td><td><code>side_effect</code></td></tr><tr><td><code>bh_nationality_country_id</code></td><td><code>user</code></td><td><code>country</code></td></tr><tr><td><code>bh_residence_country_id</code></td><td><code>user</code></td><td><code>country</code></td></tr><tr><td><code>bh_country_id</code></td><td><code>user_address</code></td><td><code>country</code></td></tr><tr><td><code>bh_source_business_id</code></td><td><code>user_credit</code></td><td><code>business</code></td></tr><tr><td><code>bh_use_id</code></td><td><code>user_medication</code></td><td><code>use</code></td></tr><tr><td><code>bh_plan_variant_id</code></td><td><code>user_subscription</code></td><td><code>subscription_plan_variant</code></td></tr><tr><td><code>bh_business_seller_id</code></td><td><code>basket</code></td><td><code>business</code></td></tr><tr><td><code>bh_business_brand_id</code></td><td><code>basket</code></td><td><code>business</code></td></tr><tr><td><code>bh_order_id</code></td><td><code>basket</code></td><td><code>order</code></td></tr><tr><td><code>bh_product_variant_id</code></td><td><code>basket_item</code></td><td><code>product_variant</code></td></tr><tr><td><code>bh_business_id</code></td><td><code>business_snapshot</code></td><td><code>business_id</code></td></tr></tbody></table>

{% hint style="info" %}
The `bh_` prefix stands for BudHub and makes it immediately clear which fields reference external data. When reading the schema, any field with this prefix requires a Gateway call to resolve the full record.
{% endhint %}

***

## Middlewares

Toke has two middlewares that control access to its endpoints:

| Middleware              | Applied to                      | Purpose                                                                    |
| ----------------------- | ------------------------------- | -------------------------------------------------------------------------- |
| `auth_middleware`       | All endpoints                   | Validates either a `toke_token` (direct) or `gateway_token` (from Gateway) |
| `auth_middleware_admin` | Admin-restricted endpoints only | Ensures the Gateway caller has `main_type: Admin`                          |

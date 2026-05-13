---
icon: memo-pad
---

# Reviews

Review submission, listing, voting, and moderation. Business reviews trigger a credit flow that mints a `user_credit` in Toke and a `business_credit` in BudHub, gated by the user's onboarding state.

## Overview

Reviews are stored in Toke. Each review targets either a `business` or a `product` in BudHub, identified by `bh_reviewable_id` together with `reviewable_type`.

Two side effects run automatically when a review is created:

* **Rating recalculation.** Business reviews update `avg_rating` and `review_count` on the local `business_snapshot` table and on the BudHub `business_admin_table` via the Gateway. Product reviews update the corresponding product record in BudHub.
* **Credit flow.** Only the first business review a user leaves for a given business qualifies for a credit. If the user has completed onboarding, the credit is granted immediately. Otherwise, a `pending_credit` row is created and processed when onboarding completes.

The credit flow keeps both sides of the marketplace in sync: a Toke `user_credit` and a BudHub `business_credit` are created in a single logical operation, with cross-references stored on both rows for reconciliation.

## Flow Overview

```
[Frontend]                  [Toke]                      [BudHub via Gateway]
    │                          │                                │
    ├─ POST /review ──────────▶│                                │
    │                          │                                │
    │                          ├─ check_review_credit           │
    │                          │   ├─ qualifies? + onboarding?  │
    │                          │   │     ├─ YES → create_review_credit
    │                          │   │     │           ├─ user_credit
    │                          │   │     │           └─ business_credit ─────▶│
    │                          │   │     └─ NO  → pending_credit              │
    │                          │   │                                          │
    │                          ├─ update_business_rating ────────────────────▶│
    │                          │                                              │ (avg + count)
    │                          │                                              │
    ├─ POST /onboarding/complete                                               │
    │                          ├─ check_pending_credit                        │
    │                          │   └─ for each pending → create_review_credit │
    │                          │                                              │
    ◀──────────────────────────│                                              │
```

## How data flows between apps

| Caller           | Destination | Path                                                                                                        | Auth         |
| ---------------- | ----------- | ----------------------------------------------------------------------------------------------------------- | ------------ |
| Frontend         | Toke        | `POST /review`, `GET /business_review`, `GET /user_reviews`, `POST /review/upvote`, `POST /review/downvote` | `toke_token` |
| Frontend (admin) | Toke        | `POST /review/status`                                                                                       | `toke_token` |
| Toke function    | BudHub      | Gateway `POST /business_credit`                                                                             | Service Key  |
| Toke function    | BudHub      | Gateway `POST /update_business_admin_table`                                                                 | Service Key  |

## Endpoints

### POST /review

Creates a review on a product or a business. Triggers downstream rating recalculation and, for business reviews, the credit flow.

**Auth:** `toke_token`

**Headers:**

| Header          | Required | Value                 |
| --------------- | -------- | --------------------- |
| `Authorization` | Yes      | `Bearer <toke_token>` |
| `Content-Type`  | Yes      | `application/json`    |
| `X-Data-Source` | Yes      | `test` or `live`      |
| `X-Branch`      | Yes      | `dev` or `v1`         |

**Body:**

| Field              | Type    | Required | Description                                         |
| ------------------ | ------- | -------- | --------------------------------------------------- |
| `user_id`          | Integer | Yes      | Must match `$auth.id` — endpoint rejects mismatches |
| `reviewable_type`  | Enum    | Yes      | `product` or `business`                             |
| `bh_reviewable_id` | Integer | Yes      | BudHub id of the target (business or product)       |
| `overall_rating`   | Decimal | Yes      | Typically 1-5                                       |
| `title`            | Text    | No       | Trimmed                                             |
| `body`             | Text    | No       | Trimmed                                             |
| `country_id`       | Integer | No       | Stored as `bh_country_id` on the review row         |

**Example request:**

```json
{
  "user_id": 17,
  "reviewable_type": "business",
  "bh_reviewable_id": 90,
  "overall_rating": 4,
  "title": "Great experience",
  "body": "Friendly staff and fast service.",
  "country_id": 209
}
```

**Response**

```json
{
  "id": 312,
  "created_at": 1778504951815,
  "user_id": 17,
  "reviewable_type": "business",
  "bh_reviewable_id": 90,
  "overall_rating": 4,
  "title": "Great experience",
  "body": "Friendly staff and fast service.",
  "reviewer_type": "Non-verified",
  "publish_date": 1778504951815,
  "status": "Published",
  "is_public": true,
  "bh_country_id": 209,
  "upvote_user_ids": [],
  "downvote_user_ids": []
}
```

**Errors:**

| Status | Message       | When                                        |
| ------ | ------------- | ------------------------------------------- |
| `401`  | Invalid Token | `user_id` in body does not match `$auth.id` |

#### Flow

1. Validate that `user_id == $auth.id`
2. Insert the `review` row with defaults: `status = "Published"`, `is_public = true`, `reviewer_type = "Non-verified"`, `publish_date = now`
3. If `reviewable_type == "product"`, call `products/update_product_reviews` (BudHub) via Gateway
4. If `reviewable_type == "business"`:
   * Call `reviews/check_review_credit` to evaluate credit eligibility
   * Call `reviews/update_business_rating` to recalculate `avg_rating` and `review_count`

{% hint style="info" %}
The credit logic only triggers for business reviews. Product reviews do not generate credits for now.
{% endhint %}

***

### GET /business\_review

Lists published, public reviews for a given business with optional filters and dynamic sorting. Built for the business profile / review tab UI.

**Auth:** `toke_token`

**Headers:**

| Header          | Required | Value                 |
| --------------- | -------- | --------------------- |
| `Authorization` | Yes      | `Bearer <toke_token>` |
| `Content-Type`  | No       | `application/json`    |
| `X-Data-Source` | Yes      | `test` or `live`      |
| `X-Branch`      | Yes      | `dev` or `v1`         |

**Query parameters:**

| Field           | Type    | Required | Description                                                         |
| --------------- | ------- | -------- | ------------------------------------------------------------------- |
| `business_id`   | Integer | Yes      | BudHub business id (maps to `bh_reviewable_id`)                     |
| `rating_min`    | Integer | No       | Inclusive lower bound on `overall_rating`                           |
| `rating_max`    | Integer | No       | Inclusive upper bound on `overall_rating`                           |
| `upvotes_min`   | Integer | No       | Inclusive lower bound on upvote count (length of `upvote_user_ids`) |
| `upvotes_max`   | Integer | No       | Inclusive upper bound on upvote count                               |
| `downvotes_min` | Integer | No       | Inclusive lower bound on downvote count                             |
| `downvotes_max` | Integer | No       | Inclusive upper bound on downvote count                             |
| `sort_field`    | Text    | No       | Column name to sort by. Defaults to `id`.                           |
| `sort_asc`      | Boolean | No       | `true` for ascending, `false` or omitted for descending             |

**Example request:**

```
GET /business_review?business_id=90&rating_min=4&sort_field=overall_rating&sort_asc=false
```

**Response**

```json
[
  {
    "id": 312,
    "created_at": 1778504951815,
    "user_id": 17,
    "reviewable_type": "business",
    "bh_reviewable_id": 90,
    "overall_rating": 4,
    "title": "Great experience",
    "body": "Friendly staff and fast service.",
    "status": "Published",
    "is_public": true,
    "upvote_user_ids": [22, 41],
    "downvote_user_ids": []
  }
]
```

**Errors:**

| Status | Message      | When                          |
| ------ | ------------ | ----------------------------- |
| `401`  | Unauthorized | Missing or invalid auth token |

Notes:

* Only reviews with `reviewable_type == "business"` and `status == "Published"` are returned
* Empty or missing filter values are skipped (no filter applied for that field)
* Upvote and downvote ranges filter by array length on `upvote_user_ids` and `downvote_user_ids` respectively

***

### GET /user\_reviews

Lists all reviews authored by the authenticated user, sorted by most recent first.

**Auth:** `toke_token`

**Headers:**

| Header          | Required | Value                 |
| --------------- | -------- | --------------------- |
| `Authorization` | Yes      | `Bearer <toke_token>` |
| `Content-Type`  | No       | `application/json`    |
| `X-Data-Source` | Yes      | `test` or `live`      |
| `X-Branch`      | Yes      | `dev` or `v1`         |

**Query parameters:**

| Field     | Type    | Required | Description           |
| --------- | ------- | -------- | --------------------- |
| `user_id` | Integer | Yes      | Must match `$auth.id` |

**Example request:**

```
GET /user_reviews?user_id=17
```

**Response**

```json
[
  {
    "id": 312,
    "created_at": 1778504951815,
    "user_id": 17,
    "reviewable_type": "business",
    "bh_reviewable_id": 90,
    "overall_rating": 4,
    "status": "Published"
  }
]
```

**Errors:**

| Status | Message       | When                                         |
| ------ | ------------- | -------------------------------------------- |
| `401`  | Invalid Token | `user_id` in query does not match `$auth.id` |

***

### POST /review/upvote

Toggles the caller's upvote on a review. Sending the same request twice adds then removes the upvote.

**Auth:** `toke_token`

**Headers:**

| Header          | Required | Value                 |
| --------------- | -------- | --------------------- |
| `Authorization` | Yes      | `Bearer <toke_token>` |
| `Content-Type`  | Yes      | `application/json`    |
| `X-Data-Source` | Yes      | `test` or `live`      |
| `X-Branch`      | Yes      | `dev` or `v1`         |

**Body:**

| Field       | Type    | Required | Description           |
| ----------- | ------- | -------- | --------------------- |
| `user_id`   | Integer | Yes      | Must match `$auth.id` |
| `review_id` | Integer | Yes      | Review being voted on |

**Example request:**

```json
{
  "user_id": 17,
  "review_id": 312
}
```

**Response**

```json
true
```

A boolean indicating the new state: `true` if the upvote is now active, `false` if it was just removed.

**Errors:**

| Status | Message       | When                                |
| ------ | ------------- | ----------------------------------- |
| `401`  | Invalid Token | `user_id` does not match `$auth.id` |

{% hint style="info" %}
Upvotes and downvotes are tracked independently. A user can technically hold both on the same review at once. If exclusive voting is required, enforce it client-side or extend this endpoint to clear the opposite vote.
{% endhint %}

***

### POST /review/downvote

Toggles the caller's downvote on a review. Mirror of `/review/upvote`.

**Auth:** `toke_token`

**Headers:**

| Header          | Required | Value                 |
| --------------- | -------- | --------------------- |
| `Authorization` | Yes      | `Bearer <toke_token>` |
| `Content-Type`  | Yes      | `application/json`    |
| `X-Data-Source` | Yes      | `test` or `live`      |
| `X-Branch`      | Yes      | `dev` or `v1`         |

**Body:**

| Field       | Type    | Required | Description           |
| ----------- | ------- | -------- | --------------------- |
| `user_id`   | Integer | Yes      | Must match `$auth.id` |
| `review_id` | Integer | Yes      | Review being voted on |

**Example request:**

```json
{
  "user_id": 17,
  "review_id": 312
}
```

**Response**

```json
true
```

A boolean indicating the new state: `true` if the downvote is now active, `false` if it was just removed.

**Errors:**

| Status | Message       | When                                |
| ------ | ------------- | ----------------------------------- |
| `401`  | Invalid Token | `user_id` does not match `$auth.id` |

***

### POST /review/status

Admin-only endpoint to change a review's moderation status. Triggers a business rating recalculation if the review targets a business.

**Auth:** `toke_token`

**Headers:**

| Header          | Required | Value                 |
| --------------- | -------- | --------------------- |
| `Authorization` | Yes      | `Bearer <toke_token>` |
| `Content-Type`  | Yes      | `application/json`    |
| `X-Data-Source` | Yes      | `test` or `live`      |
| `X-Branch`      | Yes      | `dev` or `v1`         |

**Body:**

| Field       | Type    | Required | Description                                                                                         |
| ----------- | ------- | -------- | --------------------------------------------------------------------------------------------------- |
| `review_id` | Integer | Yes      | Review being moderated                                                                              |
| `status`    | Enum    | Yes      | One of: `Pending`, `Published`, `Un-Published`, `Un-Published (taken down)`, `Reported`, `Rejected` |

**Example request:**

```json
{
  "review_id": 312,
  "status": "Un-Published (taken down)"
}
```

**Response**

The updated `review` row.

**Errors:**

| Status | Message      | When                   |
| ------ | ------------ | ---------------------- |
| `401`  | Unauthorized | Caller is not an admin |

#### Flow

1. Update the review's `status` field
2. If `reviewable_type == "business"`, call `reviews/update_business_rating` so the business's `avg_rating` and `review_count` reflect the new set of published reviews

## Functions

### reviews/check\_review\_credit

Determines whether a freshly created business review qualifies for a credit and dispatches accordingly: grants immediately when onboarding is complete, queues a `pending_credit` otherwise.

**Input:**

| Input            | Type    | Description                                         |
| ---------------- | ------- | --------------------------------------------------- |
| `user_id`        | Integer | Author of the review                                |
| `bh_business_id` | Integer | BudHub business being reviewed                      |
| `amount`         | Decimal | Credit amount (from `REVIEW_CREDIT_AMOUNT` env var) |

**Logic:**

1. Count the user's existing reviews for this business
2. If the count is `<= 1` (the review just created), this qualifies as the first review
3. Load the user and inspect `onboarding_complete`
4. If `onboarding_complete == true`, call `reviews/create_review_credit` to grant the credit immediately
5. If `onboarding_complete == false`, insert a `pending_credit` row with `source_type = "Business Review"` and `source_id = bh_business_id`

**Output:**

```json
[
  { "id": 312, "user_id": 17, "bh_reviewable_id": 90, "reviewable_type": "business" }
]
```

The list of qualifying reviews, used internally.

**Called by:** `POST /review` (when `reviewable_type == "business"`)

***

### reviews/create\_review\_credit

Mints the credit on both sides of the marketplace. Creates a `user_credit` in Toke, creates a `business_credit` in BudHub via the Gateway, updates the user's `credit_total`, and cross-references the two credit rows.

**Input:**

| Input            | Type    | Description                                |
| ---------------- | ------- | ------------------------------------------ |
| `user_id`        | Integer | Recipient of the user\_credit              |
| `bh_business_id` | Integer | BudHub business that originated the credit |
| `amount`         | Decimal | Credit amount                              |

**Logic:**

1. Load the user record (needed for `currency_id` and `credit_total`)
2. Insert `user_credit` with `status = "Received"`, `source = "Business Review"`, `bh_source_business_id = bh_business_id`
3. Call Gateway `POST /business_credit` with the user id, business id, amount, source label, and the new `user_credit.id`
4. Increment `user.credit_total` by `amount`
5. Update the `user_credit` row with `bh_counterpart_credit_id` pointing to the newly created `business_credit`

**Output:**

```json
{
  "id": 88,
  "user_id": 17,
  "amount": 5.00,
  "currency_id": 1,
  "status": "Received",
  "source": "Business Review",
  "bh_source_business_id": 90,
  "bh_counterpart_credit_id": 412
}
```

**Called by:** `reviews/check_review_credit`, `reviews/check_pending_credit`

***

### reviews/update\_business\_rating

Recalculates `avg_rating` and `review_count` for a business based on published, public reviews. Updates both the local `business_snapshot` row in Toke and the `business_admin_table` row in BudHub via the Gateway.

**Input:**

| Input            | Type    | Description                    |
| ---------------- | ------- | ------------------------------ |
| `bh_business_id` | Integer | BudHub business to recalculate |

**Logic:**

1. Count reviews where `reviewable_type == "business"`, `bh_reviewable_id == bh_business_id`, `status == "Published"`, `is_public == true`
2. If count > 0, sum all `overall_rating` values and divide by count, rounded to 1 decimal
3. Call Gateway `POST /update_business_admin_table` with `average_rating` and `reviews_count`
4. If the local `business_snapshot` row exists, update its `avg_rating` and `review_count`

**Output:**

```json
{
  "avg_rating": 4.3,
  "review_count": 28
}
```

**Called by:** `POST /review` (business path), `POST /review/status` (business path)

***

### reviews/check\_pending\_credit

Drains a user's queued `pending_credit` rows when onboarding completes. Each pending row spawns a `user_credit` + `business_credit` pair and is marked processed.

**Input:**

| Input     | Type    | Description                                    |
| --------- | ------- | ---------------------------------------------- |
| `user_id` | Integer | User whose pending credits should be processed |

**Logic:**

1. Query `pending_credit` rows for the user where `processed_at IS NULL` and `source_type == "Business Review"`
2. For each pending row:
   * Call `reviews/create_review_credit` with the user id, `source_id` (business id), and `amount`
   * Update the `pending_credit` row with `processed_at = now` and `resulting_credit_id = <new user_credit.id>`

**Output:**

The list of processed `pending_credit` rows.

**Called by:** `POST /onboarding/complete` (when the user transitions to `onboarding_complete == true`)

{% hint style="warning" %}
The query in this function must filter by `user_id`, `processed_at IS NULL`, AND `source_type` together. Operator precedence on combined `&&` / `||` expressions can silently widen the result set to all unprocessed business-review credits across all users. Verify the `where` clause uses `&&` between all three conditions.
{% endhint %}

## Ecosystem Flow: Business Review Credit Lifecycle

The full lifecycle of a credit earned by leaving a business review, from review submission to fulfillment in both apps.

{% stepper %}
{% step %}
**User submits a business review**

Frontend calls `POST /review` with `reviewable_type = "business"`. The review row is inserted with `status = "Published"` immediately.
{% endstep %}

{% step %}
**Eligibility check**

`reviews/check_review_credit` confirms this is the user's first review for this business. Reviews of any status count for eligibility — a previously moderated review still counts as "already reviewed".
{% endstep %}

{% step %}
**Branch on onboarding state**

* If `user.onboarding_complete == true` → proceed to immediate fulfillment.
* If `user.onboarding_complete == false` → a `pending_credit` row is created with `source_type = "Business Review"` and `source_id = bh_business_id`. Flow pauses here until onboarding completes.
{% endstep %}

{% step %}
**Onboarding completes (deferred path only)**

When the user later completes onboarding via `POST /onboarding/complete`, `reviews/check_pending_credit` drains their queued pending credits and runs the fulfillment step for each.
{% endstep %}

{% step %}
**Credit minted on both sides**

`reviews/create_review_credit` runs:

* Toke: `user_credit` row inserted with `status = "Received"` and `source = "Business Review"`
* BudHub: Gateway call creates a `business_credit` row tied to the same event
* The two rows are cross-referenced via `bh_counterpart_credit_id` and `tk_user_credit_id`
* `user.credit_total` is incremented
{% endstep %}

{% step %}
**Result**

The user sees the credit in their wallet. The business sees a matching `business_credit` in BudHub. Both records reference each other for audit.
{% endstep %}
{% endstepper %}

## Ecosystem Flow: Business Rating Recalculation

A simpler lifecycle that runs whenever a review's status changes in a way that affects the public review set.

{% stepper %}
{% step %}
**Trigger**

`reviews/update_business_rating` is called from:

* `POST /review` (when `reviewable_type == "business"`)
* `POST /review/status` (when the moderated review is a business review)
{% endstep %}

{% step %}
**Recalculate**

Count published, public reviews for the business. Sum their `overall_rating` and divide by count, rounded to 1 decimal.
{% endstep %}

{% step %}
**Propagate**

* Update `business_snapshot` in Toke
* Gateway call updates `business_admin_table.average_rating` and `reviews_count` in BudHub
{% endstep %}

{% step %}
**Result**

Both Toke and BudHub reflect the new aggregate rating. Frontend reads from either depending on the surface.
{% endstep %}
{% endstepper %}

## Independent up/down votes

Upvotes and downvotes are stored on the `review` row as two separate integer arrays (`upvote_user_ids` and `downvote_user_ids`). Toggling one does not affect the other.

This means a user could theoretically hold an active upvote AND an active downvote on the same review at once. The endpoints do not enforce mutual exclusion. Two options if exclusive voting is required:

1. **Client-side**: the frontend prevents the second vote by clearing the opposite one before sending
2. **Server-side**: extend the upvote/downvote endpoints to remove the user's id from the opposite array as part of the toggle

The choice depends on whether vote integrity needs to be guaranteed against direct API calls or only the UI.

---
icon: coin-blank
---

# Credits

How credits are created, queued, and retrieved in the Toke consumer app. Credits represent rewards earned by users through referrals or business interactions.

## Overview

Credits are stored in Toke's local `user_credit` table. Each credit has a source, either a referring user (`source_user_id`) or a business (`bh_source_business_id`). When credits are fetched, the endpoint enriches each one with a normalised `source_details` object so the Flutter client can render both source types with the same component.

Business data is read from the local `business_snapshot` table (not fetched live from BudHub), keeping this endpoint fast. User data is read directly from the local `user` table.

This page covers the read API and the high-level shape of each credit lifecycle. The detailed rules and step-by-step flows for **creating** credits live with the feature that originates each source — see the links in [Credit sources](credits.md#credit-sources).

## Credit sources

The `user_credit.source` enum has one value per kind of event that can generate a credit. Each source has its own creation flow, documented with the feature that owns it. This page handles the read side; the write side lives in the linked docs.

| Source value      | Origin event                                                     |
| ----------------- | ---------------------------------------------------------------- |
| `Friend Referred` | A referred user completes the conditions of the referral program |
| `Welcome Credit`  | A new user signs up and meets the welcome criteria               |
| `Business Review` | A user leaves their first review for a business                  |

## Credit creation

Credits are never created from the Credits feature itself. The feature that originates each source is responsible for writing the `user_credit` row (or queuing a `pending_credit` first. See below).

There are two paths a credit can take from origin event to final `user_credit` row:

**Immediate path.** The originating event happens and all activation conditions are already met. A `user_credit` is inserted directly with `status = "Received"`. The user sees it on their next read.

**Deferred path.** The originating event happens but an activation condition is still pending (the most common case: the user has not completed onboarding). A `pending_credit` row is queued and the actual `user_credit` is created later, when the condition flips.

| Source            | Activation condition                                               | When the credit row is created                                                                                                                                                                                |
| ----------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Friend Referred` | Referred friend has placed their first roder                       | Immediately on sign up of referred friend, but as pending.                                                                                                                                                    |
| `Welcome Credit`  | Using a valid referral code                                        | Immediately on sign up.                                                                                                                                                                                       |
| `Business Review` | User has completed onboarding (`user.onboarding_complete == true`) | Immediately on review submission if onboarding is complete; otherwise queued in `pending_credit` and created when onboarding completes. See [Reviews](/broken/pages/c51634e710de369b594907dc6ab489692f272d9a) |

{% hint style="info" %}
Some sources also create a matching `business_credit` in BudHub through the Gateway (currently only `Business Review`). When this happens, the `user_credit.bh_counterpart_credit_id` field links the two rows so reconciliation can run from either side of the marketplace.
{% endhint %}

## Pending credit

`pending_credit` holds credits that have been earned but cannot yet be activated. It exists so the system can capture "this user is owed something" before the conditions to grant it are met, without writing speculative rows into `user_credit`.

A pending row is created by the originating feature when the activation condition is not satisfied at the time of the event. The same feature is responsible for draining the row when the condition flips, calling the appropriate creation function and marking the pending row as processed.

Rows are never deleted. `processed_at` and `resulting_credit_id` provide a permanent audit trail of every pending credit that was ever queued, processed or not.

| Phase         | What's true                                   | Where it shows up                               |
| ------------- | --------------------------------------------- | ----------------------------------------------- |
| **Queued**    | `processed_at IS NULL`                        | Listed when querying pending credits for a user |
| **Processed** | `processed_at` set, `resulting_credit_id` set | A `user_credit` row exists with that id         |

{% hint style="info" %}
For the read API in this page, only completed `user_credit` rows are visible. Queued `pending_credit` rows are intentionally hidden from the frontend's wallet view — they represent promises, not balance. If a feature surface needs to show "credit pending activation" (e.g. during onboarding), it should query `pending_credit` directly through the originating feature's API, not through `GET /user_credit`.
{% endhint %}

## Endpoints

### GET /user\_credit

Returns a list of credits. If `user_id` is provided, validates the caller is authorised to view those credits. Each credit is enriched with source details from either a user or a business depending on the credit type.

**Auth:** `toke_token`

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

| Input     | Type    | Required | Description                                                                                      |
| --------- | ------- | -------- | ------------------------------------------------------------------------------------------------ |
| `user_id` | Integer | No       | The user whose credits to retrieve. If omitted, returns all credits (admin or service use only). |

**Example request:**

```json
{
  "user_id": 42
}
```

**Response**

```json
[
  {
    "id": 1,
    "user_id": 42,
    "amount": 10,
    "source": "Friend Referred",
    "source_user_id": 7,
    "bh_source_business_id": 0,
    "source_details": {
      "source_name": "Jane Smith",
      "source_image": "//cdn.bubble.io/.../avatar.jpg",
      "source_extra": "janesmith",
      "source_country": "Spain"
    }
  },
  {
    "id": 2,
    "user_id": 42,
    "amount": 5,
    "source": "Business Review",
    "source_user_id": 0,
    "bh_source_business_id": 17,
    "source_details": {
      "source_name": "Dispensary 9",
      "source_image": "//cdn.bubble.io/.../logo.jpg",
      "source_extra": "Dispensary",
      "source_country": "Spain"
    }
  }
]
```

{% hint style="info" %}
source\_extra refers to username for users, business\_type for businesses.
{% endhint %}

**Errors:**

| Status | Message      | When                                                                        |
| ------ | ------------ | --------------------------------------------------------------------------- |
| `401`  | Unauthorized | `user_id` doesn't match auth, caller is not admin, and no valid service key |

Notes:

* the `source_details` shape is identical regardless of whether the source is a user or a business. Flutter can render both types with the same component.
* Business data is read from the local `business_snapshot` table rather than fetching live from BudHub. Snapshot data is kept in sync via the daily sync task and real-time event pushes.

**Access control:**

The endpoint enforces one of three access conditions. If none are met, a 401 is returned.

| **Condition**                                 | **Who**                               |
| --------------------------------------------- | ------------------------------------- |
| `user_id` matches `$auth.id`                  | The user requesting their own credits |
| `main_type == "Admin"`                        | An admin user                         |
| `X-Service-Key` header matches `TOKE_APP_KEY` | Internal service-to-service call      |

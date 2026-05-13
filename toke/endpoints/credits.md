---
icon: coin-blank
---

# Credits

How credits are retrieved and enriched in the Toke consumer app. Credits represent rewards earned by users through referrals or business interactions.

## Overview

Credits are stored in Toke's local `user_credit` table. Each credit has a source, either a referring user (`source_user_id`) or a business (`bh_source_business_id`). When credits are fetched, the endpoint enriches each one with a normalised `source_details` object so the Flutter client can render both source types with the same component.

Business data is read from the local `business_snapshot` table (not fetched live from BudHub), keeping this endpoint fast. User data is read directly from the local `user` table.



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

---
icon: memo-pad
---

# Reviews

Reviews submitted by users for businesses and products, along with the structured tags that classify them (feelings, profiles, side effects, uses, activities), the comment threads they spawn, and the moderation reports filed against them.

***

## review

Stores every review left by a user, targeting either a business or a product in BudHub. The same table backs business reviews, product reviews, votes, and moderation state.

| Field                   | Type          | Description                                                                                                                                                                |
| ----------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                    | Integer       | Auto-generated unique identifier                                                                                                                                           |
| `created_at`            | Timestamp     | When the review was created. Private visibility                                                                                                                            |
| `user_id`               | Integer       | Author of the review. References `user` table                                                                                                                              |
| `reviewable_type`       | Enum          | What kind of entity is being reviewed. Values: `product`, `business`                                                                                                       |
| `bh_reviewable_id`      | Integer       | References BudHub's `business` or `product` table by ID, depending on `reviewable_type`                                                                                    |
| `overall_rating`        | Integer       | Numeric rating, typically 1–5                                                                                                                                              |
| `title`                 | Text          | Short headline. Trimmed on input                                                                                                                                           |
| `body`                  | Text          | Long-form review text. Trimmed on input                                                                                                                                    |
| `images_url`            | Text array    | URLs of images attached to the review. Images are created with a specific endpoint                                                                                         |
| `upvote_user_ids`       | Integer array | IDs of users who upvoted. Array length is the upvote count                                                                                                                 |
| `downvote_user_ids`     | Integer array | IDs of users who downvoted. Array length is the downvote count                                                                                                             |
| `comment_count`         | Integer       | Cached count of comments on this review                                                                                                                                    |
| `report_count`          | Integer       | Cached count of times this review has been reported                                                                                                                        |
| `reviewer_type`         | Enum          | Classification of the reviewer at the time of submission. Options are `Verified Purchaser`, `Non-verified`, `Verified Reviewer`. Currently writes `Non-verified` on create |
| `publish_date`          | Timestamp     | When the review went public. Set to `now` on create                                                                                                                        |
| `status`                | Enum          | Moderation status. Values: `Pending`, `Published`, `Un-Published`, `Un-Published (taken down)`, `Reported`, `Rejected`. Defaults to `Published` on create                  |
| `is_public`             | Boolean       | Whether the review is shown in public lists. Defaults to `true`                                                                                                            |
| `bh_country_id`         | Integer       | References BudHub's `country` table by ID. Captures the user's country at review time                                                                                      |
| `review_feelings_id`    | Integer array | References the `review_feeling` table by ID                                                                                                                                |
| `review_profile_id`     | Integer array | References the `review_profile` table by ID                                                                                                                                |
| `review_side_effect_id` | Integer array | References the `review_side_effect` table by ID                                                                                                                            |
| `review_use_id`         | Integer array | References the `review_use` table by ID.                                                                                                                                   |
| `review_activity_id`    | Integer array | References the `review_activity` table by ID                                                                                                                               |
| `bh_strain_variant_id`  | Integer       | References BudHub's `strain_variant` table by ID. For cannabis product reviews                                                                                             |
| `bh_subcategory_id`     | Integer       | References BudHub's `subcategory` table by ID                                                                                                                              |

{% hint style="info" %}
Upvotes and downvotes are tracked as independent arrays. A user can hold both simultaneously — the application layer does not enforce mutual exclusion.
{% endhint %}

***

## review\_feeling

Intermediate table that links a review to one feeling tag from BudHub, along with the intensity the user assigned to it. A single review can have multiple `review_feeling` rows, one per feeling.

| Field           | Type      | Description                                               |
| --------------- | --------- | --------------------------------------------------------- |
| `id`            | Integer   | Auto-generated unique identifier                          |
| `created_at`    | Timestamp | When the tag was attached to the review                   |
| `review_id`     | Integer   | The review this tag belongs to. References `review` table |
| `bh_feeling_id` | Integer   | References BudHub's `feeling` catalog table by ID         |
| `intensity`     | Integer   | How strongly the user felt this. Typically 1–10           |

***

## review\_profile

Intermediate table linking a review to a reviewer-profile tag (the kind of user leaving the review, such as experience level or context). Same shape as `review_feeling`.

| Field           | Type      | Description                                               |
| --------------- | --------- | --------------------------------------------------------- |
| `id`            | Integer   | Auto-generated unique identifier                          |
| `created_at`    | Timestamp | When the tag was attached                                 |
| `review_id`     | Integer   | The review this tag belongs to. References `review` table |
| `bh_profile_id` | Integer   | References BudHub's `profile` catalog table by ID         |
| `intensity`     | Integer   | Strength of the profile match. Typically 1–10             |

***

## review\_side\_effect

Intermediate table linking a product review to a reported side effect with its intensity. Used only for product reviews.

| Field               | Type      | Description                                                       |
| ------------------- | --------- | ----------------------------------------------------------------- |
| `id`                | Integer   | Auto-generated unique identifier                                  |
| `created_at`        | Timestamp | When the tag was attached                                         |
| `review_id`         | Integer   | The review this side effect belongs to. References `review` table |
| `bh_side_effect_id` | Integer   | References BudHub's `side_effect` catalog table by ID             |
| `intensity`         | Integer   | How strong the side effect was. Typically 1–10                    |

***

## review\_use

Intermediate table linking a review to a use-case tag with its intensity. Captures how the reviewer used the product or business.

| Field        | Type      | Description                                                      |
| ------------ | --------- | ---------------------------------------------------------------- |
| `id`         | Integer   | Auto-generated unique identifier                                 |
| `created_at` | Timestamp | When the tag was attached                                        |
| `review_id`  | Integer   | The review this use case belongs to. References `review` table   |
| `bh_use_id`  | Integer   | References BudHub's `use` catalog table by ID                    |
| `intensity`  | Integer   | How relevant this use case was to the experience. Typically 1–10 |

***

## review\_activity

Intermediate table linking a review to an activity tag with its intensity. Captures what the reviewer was doing.

| Field            | Type      | Description                                                           |
| ---------------- | --------- | --------------------------------------------------------------------- |
| `id`             | Integer   | Auto-generated unique identifier                                      |
| `created_at`     | Timestamp | When the tag was attached                                             |
| `review_id`      | Integer   | The review this activity belongs to. References `review` table        |
| `bh_activity_id` | Integer   | References BudHub's `activity` catalog table by ID                    |
| `intensity`      | Decimal   | How relevant this activity case was to the experience. Typically 1–10 |

***

## review\_comment

Threaded comments left on a review. Supports nested replies through `parent_comment_id` pointing back to another row in the same table.

| Field               | Type      | Description                                                                    |
| ------------------- | --------- | ------------------------------------------------------------------------------ |
| `id`                | Integer   | Auto-generated unique identifier                                               |
| `created_at`        | Timestamp | When the comment was posted                                                    |
| `review_id`         | Integer   | The review this comment belongs to. References `review` table                  |
| `user_id`           | Integer   | Author of the comment. References `user` table                                 |
| `parent_comment_id` | Integer   | Self-reference to `review_comment.id` for replies. Null for top-level comments |
| `body`              | Text      | The comment text                                                               |

***

## review\_report

Moderation reports filed by users against reviews or comments. Polymorphic: `reportable_type` determines what kind of entity `reportable_id` points to.

| Field             | Type      | Description                                                                                                                              |
| ----------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `id`              | Integer   | Auto-generated unique identifier                                                                                                         |
| `created_at`      | Timestamp | When the report was filed                                                                                                                |
| `reportable_type` | Enum      | What kind of entity is being reported. Values: `Review`, `Comment`                                                                       |
| `reportable_id`   | Integer   | ID of the reported entity. Points to `review` or `review_comment` depending on `reportable_type`                                         |
| `user_id`         | Integer   | The user who filed the report. References `user` table                                                                                   |
| `reason`          | Text      | The reason for the report. Stored as text. The catalog of canonical reasons lives in `report_reason` and is shown to the user as choices |
| `details`         | Text      | Free-form additional context the reporter provided                                                                                       |
| `status`          | Enum      | Moderation status of the report. Values: `Pending`, `Reviewed`, `Dismissed`, `Resolved`. Defaults to `Pending`                           |

{% hint style="info" %}
`reportable_type` is polymorphic but constrained to two values. When `reportable_type == "Review"`, `reportable_id` is a `review.id`. When `reportable_type == "Comment"`, it is a `review_comment.id`.
{% endhint %}

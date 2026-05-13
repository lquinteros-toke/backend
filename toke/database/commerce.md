---
icon: coin-blank
---

# Commerce

Manages user subscriptions linked to BudHub plans, credit transactions, queued credits awaiting activation, and the credit balance system.

***

## user\_subscription

Links a Toke user to a subscription plan variant defined in BudHub. One active subscription per user at a time.

| Field                | Type      | Description                                                                                                       |
| -------------------- | --------- | ----------------------------------------------------------------------------------------------------------------- |
| `id`                 | Integer   | Auto-generated unique identifier                                                                                  |
| `created_at`         | Timestamp | When the subscription was created. Private visibility                                                             |
| `user_id`            | Integer   | The user who holds this subscription. References `user` table                                                     |
| `bh_plan_variant_id` | Integer   | References BudHub's `subscription_plan_variant` table. Determines the plan tier, currency, frequency, and pricing |
| `status`             | Enum      | Current state of the subscription. Values: `active`, `cancelled`, `expired`. Defaults to `active`                 |
| `started_at`         | Timestamp | When the subscription began                                                                                       |
| `expires_at`         | Timestamp | When the subscription expires. Null for lifetime plans                                                            |
| `auto_renew`         | Boolean   | Whether the subscription automatically renews at expiration. Defaults to true                                     |

{% hint style="info" %}
Subscription plans and their variants (pricing per currency and frequency) are defined and managed in BudHub by admins. Toke only stores which variant the user is currently subscribed to.
{% endhint %}

***

## user\_credit

Records individual credit transactions. Every credit earned, spent, or pending gets its own line item with full context about the source and associated parties. Cross-references the matching `business_credit` in BudHub when one was created.

| Field                      | Type      | Description                                                                                                                                                                                   |
| -------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                       | Integer   | Auto-generated unique identifier                                                                                                                                                              |
| `created_at`               | Timestamp | When the transaction occurred. Private visibility                                                                                                                                             |
| `user_id`                  | Integer   | The user who earned or spent this credit. References `user` table                                                                                                                             |
| `amount`                   | Decimal   | The monetary amount of the credit                                                                                                                                                             |
| `currency_id`              | Integer   | The currency in which this credit is issued. References Toke's `currency` table. Defaults to user country of residence currency                                                               |
| `status`                   | Enum      | Current state of this credit transaction. Values: `Pending`, `Received`, `Used`. Defaults to `Pending`                                                                                        |
| `source`                   | Enum      | Where this credit originated from. Values: `Friend Referred`, `Welcome Credit`, `Business Review`                                                                                             |
| `source_user_id`           | Integer   | ID of the user whose action triggered this credit (for example, the friend who referred someone). References `user` table                                                                     |
| `bh_source_business_id`    | Integer   | References BudHub's `business` table by ID. The business whose action triggered this credit (for example, the business that was reviewed)                                                     |
| `counterpart_credit_id`    | Integer   | ID of the related credit generated from the same event. For example, when a referral generates credits for both users, each credit record points to the other. References `user_credit` table |
| `bh_counterpart_credit_id` | Integer   | References BudHub's `business_credit` table by ID. The matching credit on the BudHub side, when the event created one (for example, business review credits)                                  |

{% hint style="info" %}
When a referral generates credits, two `user_credit` records are created: one for the referrer and one for the referred user. Each record's `counterpart_credit_id` points to the other, creating a linked pair.
{% endhint %}

{% hint style="info" %}
For credits whose source has a counterpart in BudHub (currently `Business Review`), `bh_counterpart_credit_id` links to the BudHub `business_credit` row. This allows reconciliation from either side of the marketplace.
{% endhint %}

***

## pending\_credit

Holds credits that have been earned but not yet activated, because the activation condition (typically completing onboarding) has not been met. Once the condition fires, a processing function reads pending rows, creates the corresponding `user_credit`, and marks the pending row as processed. Rows are never deleted — `processed_at` together with `resulting_credit_id` provide a full audit trail.

| Field                 | Type      | Description                                                                                                       |
| --------------------- | --------- | ----------------------------------------------------------------------------------------------------------------- |
| `id`                  | Integer   | Auto-generated unique identifier                                                                                  |
| `created_at`          | Timestamp | When the credit was queued (for example, when a business review was left during onboarding). Private visibility   |
| `user_id`             | Integer   | Recipient of the credit once activated. References `user` table                                                   |
| `source_type`         | Enum      | What kind of event earned this credit. Current values: `Business Review`                                          |
| `source_id`           | Integer   | ID of the row that originated this credit. For `Business Review`, this is the BudHub `business` id                |
| `amount`              | Decimal   | The credit amount to grant on activation. Captured at queue time so later rate changes do not apply retroactively |
| `processed_at`        | Timestamp | Null while pending. Set when the credit is activated and the corresponding `user_credit` is created               |
| `resulting_credit_id` | Integer   | Null while pending. Set to the resulting `user_credit.id` once processed. References `user_credit` table          |

{% hint style="info" %}
Processed rows are kept indefinitely. To find unprocessed pending credits for a user, filter by `user_id`, `processed_at IS NULL`, AND `source_type` together. Operator precedence on combined conditions can silently widen the result set otherwise.
{% endhint %}

***

## Credit balance on the user table

The running credit balance is maintained directly on the `user` table for fast reads. These fields are updated whenever a `user_credit` record is created or its status changes.

| Field              | Description                                                                 |
| ------------------ | --------------------------------------------------------------------------- |
| `credit_total`     | Lifetime credits earned. Only increases over time                           |
| `credit_used`      | Lifetime credits redeemed on orders. Only increases over time               |
| `credit_pending`   | Credits waiting to clear, such as pending referral purchase confirmations   |
| `credit_available` | **Not stored.** Calculated as `credit_total - credit_used - credit_pending` |

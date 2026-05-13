---
icon: circle-dollar
---

# Commerce

Manages user subscriptions linked to BudHub plans, credit transactions, and the credit balance system.

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

Records individual credit transactions. Every credit earned, spent, or pending gets its own line item with full context about the source and associated parties.

| Field                   | Type      | Description                                                                                                                                                                                   |
| ----------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                    | Integer   | Auto-generated unique identifier                                                                                                                                                              |
| `created_at`            | Timestamp | When the transaction occurred. Private visibility                                                                                                                                             |
| `user_id`               | Integer   | The user who earned or spent this credit. References `user` table                                                                                                                             |
| `amount`                | Decimal   | The monetary amount of the credit                                                                                                                                                             |
| `currency_id`           | Integer   | The currency in which this credit is issued. References Toke's `currency` table. Defaults to user country of residence currency.                                                              |
| `status`                | Enum      | Current state of this credit transaction. Values: `Pending`, `Received`, `Used`. Defaults to `Pending`                                                                                        |
| `source`                | Enum      | Where this credit originated from. Values: `Friend Referred`, `Welcome Credit`                                                                                                                |
| `source_user_id`        | Integer   | ID of the user whose action triggered this credit (for example, the friend who referred someone). References `user` table                                                                     |
| `bh_source_business_id` | Integer   | BudHub ID of the business whose action triggered this credit, if applicable                                                                                                                   |
| `counterpart_credit_id` | Integer   | ID of the related credit generated from the same event. For example, when a referral generates credits for both users, each credit record points to the other. References `user_credit` table |

{% hint style="info" %}
When a referral generates credits, two `user_credit` records are created: one for the referrer and one for the referred user. Each record's `counterpart_credit_id` points to the other, creating a linked pair.
{% endhint %}

***

## Credit balance on the user table

The running credit balance is maintained directly on the `user` table for fast reads. These fields are updated whenever a `user_credit` record is created or its status changes.

| Field                 | Description                                                                 |
| --------------------- | --------------------------------------------------------------------------- |
| `credit_total`        | Lifetime credits earned. Only increases over time                           |
| `credit_used`         | Lifetime credits redeemed on orders. Only increases over time               |
| `credit_pending`      | Credits waiting to clear, such as pending referral purchase confirmations   |
| **credit\_available** | **Not stored.** Calculated as `credit_total - credit_used - credit_pending` |

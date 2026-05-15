---
icon: circle-sterling
---

# Commerce

Manages customer subscription plans with per-country pricing and fees, the subscription lifecycle and its audit ledger, credit transactions, queued credits awaiting activation, and the credit balance system.

## billing\_period

Defines the billing periods available for subscription plans (monthly, annual, lifetime, and any future variants). Acts as the source of truth for both UI metadata (savings badges, display labels) and Stripe price configuration (`interval`, `interval_count`). Adding a new period like quarterly or biennial is a single row insert here, with no schema changes elsewhere.

| Field                   | Type      | Description                                                                                                                                                                       |
| ----------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                    | Integer   | Auto-generated unique identifier                                                                                                                                                  |
| `created_at`            | Timestamp | When the period was defined. Private visibility                                                                                                                                   |
| `updated_at`            | Timestamp | When the row was last modified                                                                                                                                                    |
| `internal_code`         | Text      | Internal code used in code references. Trimmed and lowercased. Unique. Examples: `monthly`, `annual`, `lifetime`                                                                  |
| `display_name`          | Text      | Localized label shown to users. Examples: `Monthly`, `Annual`, `Lifetime`                                                                                                         |
| `short_label`           | Text      | Optional compact label for pricing UI. Examples: `/mo`, `/yr`                                                                                                                     |
| `duration_days`         | Integer   | Length of the period in days. Null for lifetime or non-time-bounded periods                                                                                                       |
| `is_recurring`          | Boolean   | Whether Stripe should treat this as a recurring subscription. `true` for monthly and annual, `false` for lifetime (one-time payment). Defaults to `true`                          |
| `stripe_interval`       | Text      | Stripe recurring interval. Values: `day`, `week`, `month`, `year`. Null for non-recurring periods                                                                                 |
| `stripe_interval_count` | Integer   | Stripe interval count. `1` for monthly, `3` for quarterly, `12` for annual when expressed as month-based, `1` for annual when `stripe_interval` is `year`. Null for non-recurring |
| `sort_order`            | Integer   | Display order in pricing UI. Lower numbers shown first                                                                                                                            |
| `is_active`             | Boolean   | Whether this period is available for new subscriptions. Defaults to `true`                                                                                                        |

{% hint style="info" %}
Lifetime subscriptions are implemented as Stripe one-time payments (Checkout `mode: payment`), not recurring subscriptions. The `is_recurring` flag drives that branching in the Checkout flow.
{% endhint %}

## subscription\_plan

Abstract subscription plans (for example `Premium`, `Pro`). These are universal across countries. Pricing and fees live on `subscription_plan_country`, so the same plan can have different prices and fee structures depending on the user's country.

| Field           | Type      | Description                                                                                                      |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------- |
| `id`            | Integer   | Auto-generated unique identifier                                                                                 |
| `created_at`    | Timestamp | When the plan was created. Private visibility                                                                    |
| `updated_at`    | Timestamp | When the plan was last modified                                                                                  |
| `name`          | Text      | Commercial name shown to users. Trimmed. Example: `Premium`                                                      |
| `internal_code` | Text      | Internal code used for feature checks in code. Trimmed and lowercased. Unique. Example: `premium`                |
| `description`   | Text      | Optional marketing description                                                                                   |
| `tier`          | Integer   | Hierarchical order (1, 2, 3...) used to compare plans on upgrade/downgrade operations                            |
| `features`      | JSON      | Feature flags or plan configuration objects                                                                      |
| `is_active`     | Boolean   | Whether the plan accepts new subscriptions. Defaults to `true`. Disabling does not affect existing subscriptions |
| `sort_order`    | Integer   | Display order in pricing UI. Lower numbers shown first                                                           |

{% hint style="info" %}
`tier` is the basis for upgrade vs downgrade detection. When a user changes plan, comparing the old plan's tier with the new one determines whether to apply the change immediately (upgrade) or schedule it for the end of the current period (downgrade).
{% endhint %}

## subscription\_plan\_country

Per-country economic context for a subscription plan. One row per (plan, country) combination. Fees here are flat for the country and do not vary by billing period, which is why they live on this table and not on `subscription_plan_price`.

| Field                    | Type      | Description                                                                                                          |
| ------------------------ | --------- | -------------------------------------------------------------------------------------------------------------------- |
| `id`                     | Integer   | Auto-generated unique identifier                                                                                     |
| `created_at`             | Timestamp | When the row was created. Private visibility                                                                         |
| `updated_at`             | Timestamp | When the row was last modified                                                                                       |
| `subscription_plan_id`   | Integer   | References Toke's `subscription_plan` table. Identifies the plan                                                     |
| `country_id`             | Integer   | References Toke's `country` table. Identifies the country, and indirectly the currency through `country.currency_id` |
| `order_fee_amount`       | Decimal   | Flat fee per order, expressed in the country's currency. Defaults to `0`                                             |
| `service_fee_percentage` | Decimal   | Percentage applied to the order total. `1.5` = 1.5%. Defaults to `0`                                                 |
| `service_fee_cap`        | Decimal   | Monetary cap on the service fee, expressed in the country's currency. Null means no cap                              |
| `is_active`              | Boolean   | Whether this plan is offered in this country. Defaults to `true`                                                     |

{% hint style="info" %}
The currency for prices and fees is resolved by following `subscription_plan_country.country_id` to `country.currency_id`. There is no `currency_id` directly on this table, the country is the single source of truth.
{% endhint %}

{% hint style="warning" %}
The combination of `subscription_plan_id` and `country_id` is unique. A plan can only have one economic context per country.
{% endhint %}

## subscription\_plan\_price

Price for a specific billing period of a plan in a country. Typically three rows per `subscription_plan_country` (monthly, annual, lifetime), but not all combinations are required. A plan can offer only annual in one country and all three in another.

| Field                          | Type      | Description                                                                                                            |
| ------------------------------ | --------- | ---------------------------------------------------------------------------------------------------------------------- |
| `id`                           | Integer   | Auto-generated unique identifier                                                                                       |
| `created_at`                   | Timestamp | When the price was created. Private visibility                                                                         |
| `updated_at`                   | Timestamp | When the price was last modified                                                                                       |
| `subscription_plan_country_id` | Integer   | References Toke's `subscription_plan_country` table. Identifies the (plan, country) combination this price applies to  |
| `billing_period_id`            | Integer   | References Toke's `billing_period` table. Identifies the period (monthly, annual, lifetime, etc.)                      |
| `amount`                       | Decimal   | Price in the country's currency, resolved through `subscription_plan_country` → `country` → `currency`                 |
| `stripe_price_id`              | Text      | Stripe Price ID (`price_...`) that links this row to the corresponding product in Stripe. Unique                       |
| `is_active`                    | Boolean   | Whether this price is offered. Defaults to `true`. Disabling does not affect existing subscriptions tied to this price |

{% hint style="warning" %}
The combination of `subscription_plan_country_id` and `billing_period_id` is unique. A given plan in a given country can only have one price per period.
{% endhint %}

{% hint style="info" %}
`stripe_price_id` is never exposed to the client. The checkout flow accepts only the local `subscription_plan_price_id` and resolves the Stripe price server-side.
{% endhint %}

## subscription\_plan\_benefit

One row per benefit line shown inside a plan card. Ordered, with optional highlighting for benefits new to this tier.

| Field                  | Type      | Description                                                                                                       |
| ---------------------- | --------- | ----------------------------------------------------------------------------------------------------------------- |
| `id`                   | Integer   | Auto-generated unique identifier                                                                                  |
| `created_at`           | Timestamp | When the price was created. Private visibility                                                                    |
| `updated_at`           | Timestamp | When the row was last modified                                                                                    |
| `subscription_plan_id` | Integer   | The plan this benefit belongs to. References Toke's `subscription_plan` table                                     |
| `label`                | Text      | The benefit text. Supports inline `**bold**` for emphasized phrases                                               |
| `is_highlighted`       | Boolean   | If `true`, rendered with orange checkmark and bold text (used for benefits new in this tier). Defaults to `false` |
| `sort_order`           | Integer   | Display order within the plan card. Lower numbers shown first                                                     |
| `is_active`            | Boolean   | Whether the benefit is shown. Defaults to `true`                                                                  |

## user\_subscription

A user's active or historical subscription. A user can have many rows over time but typically only one is `active` at any moment. Several fields are denormalized from `subscription_plan_price` so that order-time lookups (for fees and tier) avoid a 3-hop join.

| Field                                  | Type      | Description                                                                                                                                                                                   |
| -------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                   | Integer   | Auto-generated unique identifier                                                                                                                                                              |
| `created_at`                           | Timestamp | When the subscription was created. Private visibility                                                                                                                                         |
| `updated_at`                           | Timestamp | When the row was last modified                                                                                                                                                                |
| `user_id`                              | Integer   | The user who holds this subscription. References `user` table                                                                                                                                 |
| `subscription_plan_price_id`           | Integer   | References Toke's `subscription_plan_price` table. The specific (plan, country, period) the user signed up for                                                                                |
| `subscription_plan_id`                 | Integer   | Denormalized from `subscription_plan_price` → `subscription_plan_country` → `subscription_plan`. References Toke's `subscription_plan` table. Used for fast tier lookups                      |
| `subscription_plan_country_id`         | Integer   | Denormalized from `subscription_plan_price` → `subscription_plan_country`. References Toke's `subscription_plan_country` table. Used for fast fee lookups                                     |
| `billing_period_id`                    | Integer   | Denormalized from `subscription_plan_price.billing_period_id`. References Toke's `billing_period` table                                                                                       |
| `status`                               | Enum      | Current lifecycle state. Values: `incomplete`, `trialing`, `active`, `past_due`, `paused`, `canceled`, `expired`                                                                              |
| `started_at`                           | Timestamp | When the subscription was first activated. Defaults to creation time                                                                                                                          |
| `current_period_start`                 | Timestamp | Start of the current billing period (matches Stripe `current_period_start`). Updated on each renewal                                                                                          |
| `current_period_end`                   | Timestamp | End of the current billing period. Null for lifetime subscriptions                                                                                                                            |
| `expires_at`                           | Timestamp | When access expires. Null means no end (lifetime or active rolling). Set to `current_period_end` when the user cancels at period end                                                          |
| `cancel_at_period_end`                 | Boolean   | If `true`, the subscription ends at `current_period_end` and does not renew. Defaults to `false`                                                                                              |
| `canceled_at`                          | Timestamp | When the user requested cancellation. Independent of when access actually ends                                                                                                                |
| `trial_ends_at`                        | Timestamp | When the trial period ends (if any). Null for subscriptions without a trial                                                                                                                   |
| `last_payment_at`                      | Timestamp | When the most recent successful payment was recorded                                                                                                                                          |
| `subscription_plan_price_id_scheduled` | Integer   | References Toke's `subscription_plan_price` table. The price the user will move to at `scheduled_change_at`. Used for downgrades (scheduled to end of period). Null when no change is pending |
| `scheduled_change_at`                  | Timestamp | When the scheduled price change applies. Typically equals `current_period_end`                                                                                                                |
| `external_subscription_id`             | Text      | Stripe subscription ID (`sub_...`) for recurring subscriptions. Null for lifetime (one-time payment). Unique                                                                                  |
| `external_customer_id`                 | Text      | Stripe customer ID (`cus_...`). Persisted across subscriptions so the same user reuses the same Stripe customer                                                                               |
| `metadata`                             | JSON      | Free-form metadata for promotions, attribution, or admin notes                                                                                                                                |

{% hint style="info" %}
`expires_at` semantics:

* Null + status `active` means lifetime or active rolling (auto-renew).
* Set + status `active` + `cancel_at_period_end = true` means canceled but still accessible until that date.
* Set + status `expired` means access has ended.
{% endhint %}

{% hint style="warning" %}
Plan changes use two different paths. Upgrades apply immediately and Stripe prorates the difference. Downgrades are scheduled: the new price is written to `subscription_plan_price_id_scheduled` and `scheduled_change_at` is set to `current_period_end`. A daily background task applies the change on that date.
{% endhint %}

## user\_subscription\_event

Append-only ledger of subscription state changes. Every mutation on `user_subscription` writes a paired row here for full audit history and Stripe webhook idempotency. Rows are never updated or deleted.

| Field                  | Type      | Description                                                                                                                                                                                                                                                                                                    |
| ---------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                   | Integer   | Auto-generated unique identifier                                                                                                                                                                                                                                                                               |
| `created_at`           | Timestamp | When the event was recorded. Private visibility                                                                                                                                                                                                                                                                |
| `user_subscription_id` | Integer   | References Toke's `user_subscription` table. The subscription this event applies to                                                                                                                                                                                                                            |
| `event_type`           | Enum      | What happened. Values: `created`, `trial_started`, `activated`, `renewed`, `upgraded`, `downgraded`, `downgrade_scheduled`, `downgrade_applied`, `scheduled_change_canceled`, `canceled`, `cancel_scheduled`, `reactivated`, `paused`, `resumed`, `expired`, `payment_succeeded`, `payment_failed`, `refunded` |
| `source`               | Enum      | Where the event originated. Values: `stripe_webhook`, `user_action`, `admin_action`, `system`                                                                                                                                                                                                                  |
| `amount_paid`          | Decimal   | Only set on payment events (`payment_succeeded`, `refunded`, etc.). Captured at event time so historical reporting is not affected by later price changes                                                                                                                                                      |
| `currency_id`          | Integer   | References Toke's `currency` table. The currency the payment was made in                                                                                                                                                                                                                                       |
| `previous_status`      | Text      | The `user_subscription.status` value before the event                                                                                                                                                                                                                                                          |
| `new_status`           | Text      | The `user_subscription.status` value after the event                                                                                                                                                                                                                                                           |
| `external_event_id`    | Text      | Stripe event ID. Used as an idempotency key so duplicate webhook deliveries do not create duplicate rows. Unique                                                                                                                                                                                               |
| `metadata`             | JSON      | Free-form event payload. Examples: from/to price IDs for upgrades, Stripe invoice ID for payments, admin reason for manual overrides                                                                                                                                                                           |

{% hint style="warning" %}
The unique constraint on `external_event_id` is critical. Stripe retries webhooks aggressively, and this index ensures duplicate inserts fail at the database level rather than being caught by application logic.
{% endhint %}

{% hint style="info" %}
`amount_paid` and `previous_status` / `new_status` are snapshots, not live joins. The ledger's role is to be a frozen photograph of what happened, even if the related subscription or plan is later modified.
{% endhint %}

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

## pending\_credit

Holds credits that have been earned but not yet activated, because the activation condition (typically completing onboarding) has not been met. Once the condition fires, a processing function reads pending rows, creates the corresponding `user_credit`, and marks the pending row as processed. Rows are never deleted, `processed_at` together with `resulting_credit_id` provide a full audit trail.

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

## Credit balance on the user table

The running credit balance is maintained directly on the `user` table for fast reads. These fields are updated whenever a `user_credit` record is created or its status changes.

| Field              | Description                                                                 |
| ------------------ | --------------------------------------------------------------------------- |
| `credit_total`     | Lifetime credits earned. Only increases over time                           |
| `credit_used`      | Lifetime credits redeemed on orders. Only increases over time               |
| `credit_pending`   | Credits waiting to clear, such as pending referral purchase confirmations   |
| `credit_available` | **Not stored.** Calculated as `credit_total - credit_used - credit_pending` |

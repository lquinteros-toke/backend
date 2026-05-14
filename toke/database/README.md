---
icon: database
---

# Database

Toke's database holds consumer-specific data organized into five functional areas. Each area groups related tables that work together to support a specific part of the application.

***

## Tables by category

### Users

Core user accounts, personal information, delivery addresses, and health profiles.

| Table                                        | Description                                                                                      |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| [user](users.md)                             | Central user record with identity, preferences, onboarding progress, credits, and account status |
| [user\_address](users.md#user_address)       | Physical addresses for delivery and billing. Multiple addresses per user                         |
| [user\_health](users.md#user_health)         | Optional health profile with physical measurements, lifestyle habits, and medical conditions     |
| [user\_medication](users.md#user_medication) | Medications the user is currently taking, with dosage and frequency                              |

### Reviews

Product and dispensary review system with properties, comments, votes, and moderation.

| Table                                                 | Description                                                                |
| ----------------------------------------------------- | -------------------------------------------------------------------------- |
| [review](reviews.md#review)                           | Main review record linking a user to a BudHub product or business          |
| [review\_feeling](reviews.md#review_feeling)          | Feelings experienced from a product, with intensity rating                 |
| [review\_use](reviews.md#review_use)                  | Purposes or contexts the product was used for, with intensity rating       |
| [review\_profile](reviews.md#review_profile)          | Flavor and aroma profiles identified in the product, with intensity rating |
| [review\_side\_effect](reviews.md#review_side_effect) | Negative side effects experienced, with intensity rating                   |
| [review\_comment](reviews.md#review_comment)          | User comments on reviews, supporting threaded replies                      |
| [review\_report](reviews.md#review_report)            | Reports filed against reviews or comments for moderation                   |

### Commerce

Shopping baskets, subscriptions, and credit transactions.

| Table                                                                | Description                                                                                                          |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| [billing\_period](commerce.md#billing_period)                        | Catalog of billing periods (monthly, annual, lifetime).                                                              |
| [subscription\_plan](commerce.md#subscription_plan)                  | Abstract plans (Premium, Pro). Universal across countries. No pricing here.                                          |
| [subscription\_plan\_country](commerce.md#subscription_plan_country) | Per-country economic context for a plan. Holds the fees (order fee, service fee %, cap) for a (plan, country) combo. |
| [subscription\_plan\_price](commerce.md#subscription_plan_price)     | Actual price for a (plan, country, billing period) combo.                                                            |
| [user\_subscription](commerce.md#user_subscription)                  | A user's subscription, active or historical. Tracks status, period dates, expiry, etc.                               |
| [user\_subscription\_event](commerce.md#user_subscription_event)     | Append-only audit ledger. One row per state change.                                                                  |
| [user\_credit](commerce.md#user_credit)                              | Individual credit transactions with amount, source, and status                                                       |
| [pending\_credit](commerce.md#pending_credit)                        | Credits that have been earned but not yet activated                                                                  |

### Lookups

Local reference tables used across the application.

| Table                                            | Description                                                               |
| ------------------------------------------------ | ------------------------------------------------------------------------- |
| [currency](lookups.md#currency)                  | Currencies with names, symbols, and ISO codes. Also exists in BudHub      |
| [habits\_frequency](lookups.md#habits_frequency) | Standardized frequency options for lifestyle habits in the health profile |
| [report\_reason](lookups.md#report_reason)       | Predefined reasons for flagging a review or comment                       |

### Budhub Snapshots

Information pulled from BudHub and stored in Toke

| Table              | Description                                         |
| ------------------ | --------------------------------------------------- |
| business\_snapshot | Consumer-optimized snapshot of BudHub business data |

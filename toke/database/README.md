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

| Table                                                                          | Description                                                                |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| [review](/broken/pages/96cfd469d44bf403eee03c3b9eda59d832901cb9)               | Main review record linking a user to a BudHub product or business          |
| [review\_feeling](/broken/pages/96cfd469d44bf403eee03c3b9eda59d832901cb9)      | Feelings experienced from a product, with intensity rating                 |
| [review\_use](/broken/pages/96cfd469d44bf403eee03c3b9eda59d832901cb9)          | Purposes or contexts the product was used for, with intensity rating       |
| [review\_profile](/broken/pages/96cfd469d44bf403eee03c3b9eda59d832901cb9)      | Flavor and aroma profiles identified in the product, with intensity rating |
| [review\_side\_effect](/broken/pages/96cfd469d44bf403eee03c3b9eda59d832901cb9) | Negative side effects experienced, with intensity rating                   |
| [review\_comment](/broken/pages/96cfd469d44bf403eee03c3b9eda59d832901cb9)      | User comments on reviews, supporting threaded replies                      |
| [review\_report](/broken/pages/96cfd469d44bf403eee03c3b9eda59d832901cb9)       | Reports filed against reviews or comments for moderation                   |

### Commerce

Shopping baskets, subscriptions, and credit transactions.

| Table                                                                  | Description                                                                  |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| [basket](/broken/pages/fafd7e326c5f7d1c0306ff218c9b83665f32bc79)       | Shopping basket tied to a user and a specific BudHub seller                  |
| [basket\_item](/broken/pages/fafd7e326c5f7d1c0306ff218c9b83665f32bc79) | Individual line items in a basket, each referencing a BudHub product variant |
| [user\_subscription](commerce.md#user_subscription)                    | Active subscription linking a user to a BudHub plan variant                  |
| [user\_credit](commerce.md#user_credit)                                | Individual credit transactions with amount, source, and status               |

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

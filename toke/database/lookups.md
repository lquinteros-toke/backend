---
icon: magnifying-glass
---

# Lookups

Local reference tables that support Toke's functionality. These tables exist independently in Toke and do not require Gateway calls to access.

***

## currency

List of currencies with their names, symbols, and ISO codes. This table also exists in BudHub DB with the same data. Toke's local copy is used for displaying prices and credits without needing a cross-database call.

| Field        | Type    | Description                                               |
| ------------ | ------- | --------------------------------------------------------- |
| `id`         | Integer | Auto-generated unique identifier                          |
| `created_at` | Integer | When the record was created                               |
| `name`       | Text    | Full currency name (for example, "British Pound", "Euro") |
| `symbol`     | Text    | Currency symbol (for example, `£`, `€`, `$`)              |
| `iso_code`   | Text    | ISO 4217 currency code (for example, `GBP`, `EUR`, `USD`) |

{% hint style="info" %}
Both Toke and BudHub maintain a `currency` table. The records should be kept in sync. Toke's copy avoids the need for a Gateway call every time a price or credit amount needs to be displayed.
{% endhint %}

***

## habits\_frequency

Standardized set of frequency options used in the health and lifestyle profile. Referenced by several fields in the `user_health` table including exercise frequency, alcohol consumption, smoking habit, and caffeine consumption.

| Field        | Type      | Description                                                         |
| ------------ | --------- | ------------------------------------------------------------------- |
| `id`         | Integer   | Auto-generated unique identifier                                    |
| `created_at` | Timestamp | When the record was created. Private visibility                     |
| `name`       | Text      | Frequency label (for example, "Never", "Rarely", "Weekly", "Daily") |

***

## report\_reason

Predefined reasons for flagging a review or comment. Used when creating `review_report` records. The `reason` field on a report stores the text value from this table.

| Field        | Type      | Description                                                            |
| ------------ | --------- | ---------------------------------------------------------------------- |
| `id`         | Integer   | Auto-generated unique identifier                                       |
| `created_at` | Timestamp | When the record was created. Private visibility                        |
| `name`       | Text      | Reason label (for example, "Spam", "Offensive", "Misleading", "Other") |

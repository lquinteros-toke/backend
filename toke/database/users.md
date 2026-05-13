---
icon: user
---

# Users

The `user` table is the central record for every Toke consumer. It holds identity, preferences, onboarding progress, credits, and account status. Every other Toke table references back to this one.

***

## user

Core identity and account data for every Toke consumer. Used across the entire app for authentication, profile display, order history, and reviews.

| Field                          | Type                                                                            | Description                                                                                                                                                                                            |
| ------------------------------ | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                           | Integer                                                                         | Auto-generated unique identifier                                                                                                                                                                       |
| `created_at`                   | Timestamp                                                                       | When the account was created. Private visibility. Defaults to now                                                                                                                                      |
| `email`                        | Email                                                                           | User's email address. Unique. Trimmed and lowercased. Used for login                                                                                                                                   |
| `password`                     | Password                                                                        | Hashed password. Internal visibility only. Minimum 8 characters, at least 1 letter and 1 digit                                                                                                         |
| `main_type`                    | Enum                                                                            | Controls permissions across the app. Values: `Admin`, `Customer`. Defaults to `Customer`                                                                                                               |
| `sub_type`                     | Enum                                                                            | Determines which product recommendations, content, and features the app surfaces. Values: `Recreational`, `Medical`                                                                                    |
| `first_name`                   | Text                                                                            | User's first name                                                                                                                                                                                      |
| `last_name`                    | Text                                                                            | User's last name                                                                                                                                                                                       |
| `full_name`                    | Text                                                                            | Combined first and last name                                                                                                                                                                           |
| `username`                     | Text                                                                            | Unique public display name chosen by the user. Must be unique across all Toke users                                                                                                                    |
| `d_o_b`                        | Date                                                                            | User's date of birth. Used for age verification and personalization                                                                                                                                    |
| `gender`                       | Enum                                                                            | User's gender identity. Values: `Male`, `Female`, `Non Binary`, `Prefer Not To Say`                                                                                                                    |
| `phone_number`                 | Text                                                                            | Contact phone number including country code                                                                                                                                                            |
| `bh_nationality_country_id`    | Integer                                                                         | User's nationality. References BudHub's `country` table by ID. Used for language and regulatory compliance                                                                                             |
| `bh_residence_country_id`      | Integer                                                                         | Country where the user currently lives. References BudHub's `country` table by ID. Used for currency, legal compliance, and delivery availability                                                      |
| `block_marketing`              | Boolean                                                                         | Whether the user has opted out of receiving marketing emails and promotional notifications. Defaults to false                                                                                          |
| `profile_picture_url`          | Text                                                                            | URL of the user's profile picture                                                                                                                                                                      |
| `onboarding_step`              | Integer                                                                         | Tracks the last completed onboarding step so the user can resume if they exit the app mid-flow. Defaults to 0                                                                                          |
| `onboarding_complete`          | Boolean                                                                         | Whether the user has finished the full onboarding process. Set to true after the final step is completed or skipped. Defaults to false                                                                 |
| `age_verified`                 | Boolean                                                                         | Whether the user has completed age and identity verification. Required before placing orders in most jurisdictions. Defaults to false                                                                  |
| `email_verified`               | Boolean                                                                         | Wheter the user has verified their email. DEfaults to false                                                                                                                                            |
| `referral_code`                | Text                                                                            | A unique alphanumeric code generated for each user at signup. Shared with friends to earn referral credits. Must be unique across all users                                                            |
| `used_referral_code`           | Text                                                                            | The referral code this user entered when signing up. Links to the referrer                                                                                                                             |
| `first_login`                  | Boolean                                                                         | Tracks whether the user has completed their first app session. Used to trigger onboarding flows and welcome content. Defaults to true                                                                  |
| `credit_total`                 | Decimal                                                                         | Lifetime total credits earned from referrals, promotions, and rewards. Only increases. Defaults to 0                                                                                                   |
| `credit_used`                  | Decimal                                                                         | Lifetime total credits redeemed on orders. Only increases. Defaults to 0                                                                                                                               |
| `credit_pending`               | Decimal                                                                         | Credits earned but not yet available for use, waiting to clear (for example, pending referral purchase confirmation). Defaults to 0                                                                    |
| `currency_id`                  | Integer                                                                         | The user's preferred currency for displaying prices, credits, and fees. References Toke's local `currency` table                                                                                       |
| `enrolled`                     | Boolean                                                                         | Whether the user has completed enrollment into the beta program. Defaults to false                                                                                                                     |
| `status`                       | Enum                                                                            | Current account state. Controls access to app features. Values: `Active`, `Flagged`, `Suspended`, `Blocked`. Defaults to `Active`                                                                      |
| `language`                     | Text                                                                            | User's preferred language in locale format derived from their residence country (for example, `en_gb`, `es_ar`, `de_de`). Used for UI localization and content language selection. Defaults to `en_gb` |
| `user_health_id`               | Integer                                                                         | Direct reference to the user's health profile. References `user_health` table                                                                                                                          |
| `revenue_generated`            | Date                                                                            | Total purchase value generated by this user's referrals. Updated each time a referred user places an order. Defaults to 0                                                                              |
| `residence_country_changed_at` | Timestamp                                                                       | When the residence country was last changed. Used to enforce 6-month restriction                                                                                                                       |
| `verification_code`            | Text                                                                            | A 6-digit code sent to the user's email. Cleared after successful verification                                                                                                                         |
| `verification_code_expires_at` | Timestamp                                                                       | When the code expires. Codes should be valid for 10 minutes                                                                                                                                            |
| `magic_link`                   | <p>object {<br>token: text,<br>expiration: timestamp,<br>used: boolean<br>}</p> | Stores the created maigc\_link tokens                                                                                                                                                                  |
| `reset_code`                   | Text                                                                            | The 6-character OTP code currently valid for this user.                                                                                                                                                |
| `reset_code_expires_at`        | Timestamp                                                                       | UTC timestamp after which the code is no longer accepted. Set to 15 minutes from the moment of generation.                                                                                             |

{% hint style="info" %}
**credit\_available** is not stored as a field. It is calculated as `credit_total - credit_used - credit_pending`.
{% endhint %}

***

## Naming conventions

| Field       | Values                                      | Purpose                                         |
| ----------- | ------------------------------------------- | ----------------------------------------------- |
| `main_type` | `Admin`, `Customer`                         | Access level. Same naming across JWT and BudHub |
| `sub_type`  | `Recreational`, `Medical`                   | Cannabis preference                             |
| `status`    | `Active`, `Flagged`, `Suspended`, `Blocked` | Account moderation state                        |

***

## user\_health

One record per user. Stores physical measurements, lifestyle habits, and disclosed medical conditions.

| Field                      | Type      | Description                                                                                                                                                                                        |
| -------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                       | Integer   | Auto-generated unique identifier                                                                                                                                                                   |
| `created_at`               | Timestamp | When the record was created. Private visibility                                                                                                                                                    |
| `user_id`                  | Integer   | References `user`. One-to-one relationship                                                                                                                                                         |
| `height`                   | Decimal   | User's height as a numeric value. Interpreted together with `height_unit`                                                                                                                          |
| `height_unit`              | Enum      | Unit of measurement for height. Values: `cm`, `ft`. Defaults to `cm`.                                                                                                                              |
| `weight`                   | Decimal   | User's weight as a numeric value. Interpreted together with `weight_unit`                                                                                                                          |
| `weight_unit`              | Enum      | Unit of measurement for weight. Values: `kg`, `lb`                                                                                                                                                 |
| `sex_at_birth`             | Enum      | Biological sex assigned at birth. Relevant for dosage and effect personalization since cannabinoid metabolism differs by biological sex. Values: `Male`, `Female`, `Intersex`, `Prefer Not To Say` |
| `exercise_frequency`       | Integer   | How often the user exercises. Affects metabolism-based recommendations. References `habits_frequency` table                                                                                        |
| `body_type`                | Enum      | User's self-identified body type. Values: `Ectomorph`, `Mesomorph`, `Endomorph`                                                                                                                    |
| `alcohol_consumption`      | Integer   | How often the user drinks alcohol. References `habits_frequency` table                                                                                                                             |
| `smoking_habit`            | Integer   | How often the user smokes. References `habits_frequency` table                                                                                                                                     |
| `caffeine_consumption`     | Integer   | How often the user drinks coffee or caffeinated beverages. References `habits_frequency` table                                                                                                     |
| `body_fat_percentage`      | Decimal   | User's estimated body fat percentage.                                                                                                                                                              |
| `bh_medical_condition_ids` | Integer   | Array of BudHub `use` IDs representing medical conditions or therapeutic needs the user has disclosed, such as pain relief, anxiety, or insomnia                                                   |

{% hint style="info" %}
The `medical_condition_ids` field references BudHub's `use` table.
{% endhint %}

{% hint style="info" %}
The `exercise_frequency`, `alcohol_consumption`, `smoking_habit`, and `caffeine_consumption` fields all reference the `habits_frequency` lookup table, which provides a standardized set of frequency options.
{% endhint %}

***

## user\_medication

One record per medication the user is currently taking. Multiple medications per user. Skippable during onboarding.

| Field           | Type      | Description                                                                                                                                                                                                                        |
| --------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`            | Integer   | Auto-generated unique identifier                                                                                                                                                                                                   |
| `created_at`    | Timestamp | When the record was created. Private visibility                                                                                                                                                                                    |
| `user_id`       | Integer   | References `user`                                                                                                                                                                                                                  |
| `bh_use_id`     | Integer   | The medical condition or therapeutic purpose this medication is prescribed for. References BudHub's `use` table by ID. Links the medication to the same condition taxonomy used in `user_health.medical_condition_ids` and reviews |
| `name`          | Text      | The medication's commercial or generic name as entered by the user (for example, "Ibuprofen", "Sertraline")                                                                                                                        |
| `dose_quantity` | Decimal   | The numeric amount per dose (for example, 500 for "500mg" or 10 for "10ml"). Interpreted together with `dose_unit`                                                                                                                 |
| `dose_unit`     | Enum      | The unit of measurement for each dose. Covers standard pharmaceutical units. Values: `mg`, `ml`, `g`, `mcg`, `IU`                                                                                                                  |
| `frequency`     | Text      | How often the user takes this medication. Free text to allow flexible input                                                                                                                                                        |

***

## user\_address

Stores physical addresses associated with a user. Supports multiple addresses per user for delivery and billing purposes. Each address includes contact details specific to that location.

| Field           | Type    | Description                                                                                                                 |
| --------------- | ------- | --------------------------------------------------------------------------------------------------------------------------- |
| `id`            | Integer | Auto-generated unique identifier                                                                                            |
| `created_at`    | Integer | When the address was created                                                                                                |
| `full_name`     | Text    | Full name of the person at this address. May differ from the user's own name                                                |
| `phone_number`  | Text    | Contact phone number for this specific address                                                                              |
| `bh_country_id` | Integer | Country of the address. References BudHub's `country` table by ID                                                           |
| `address_1`     | Text    | Primary street address (for example, "123 Main Street")                                                                     |
| `address_2`     | Text    | Apartment, suite, floor, or building name. Optional                                                                         |
| `city`          | Text    | City or town name                                                                                                           |
| `state`         | Text    | State, province, or region                                                                                                  |
| `post_code`     | Text    | Postal or ZIP code. Stored as text to support international formats such as UK postcodes                                    |
| `default`       | Boolean | Whether this is the user's primary address, used by default for deliveries. Only one address per user should be set to true |
| `user_id`       | Integer | References `user` table                                                                                                     |

{% hint style="warning" %}
When setting a new default address, the endpoint must first set `default = false` on the current default address before setting the new one to `true`. This ensures only one address is marked as default at any time.
{% endhint %}

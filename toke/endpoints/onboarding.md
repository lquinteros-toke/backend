---
icon: arrow-progress
---

# Onboarding

Handles user profile setup after account creation. These endpoints walk the user through personal information, address, residence country, username selection, and optionally health data. All endpoints require the user to be authenticated with their `toke_token`.

## Flow overview

```
POST /auth/signup                          Account created
POST /auth/send-verification               ↓
POST /auth/verify-email                    Email verified
    ↓
PATCH /onboarding/user                     Step 1: Personal info
    ↓
POST /onboarding/address                   Step 2: Default address
    ↓
PATCH /onboarding/user                     Step 3: Residence country
    ↓
GET /auth/check-username
PATCH /onboarding/user                     Step 4: Username
    ↓
POST /onboarding/health                    Step 5: Measurements + habits
POST /onboarding/health                    Step 6: Body composition
POST /onboarding/health                    Step 7: Medical conditions
POST /onboarding/medication                Step 7: Medications (optional)
    ↓
POST /onboarding/complete                  Done
```

***

## Steps breakdown

| Step | Screen                     | Fields                                                                                | Required for | Endpoint                                                  |
| ---- | -------------------------- | ------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------- |
| 0    | Account created            | email, password, name                                                                 | Everyone     | `POST /auth/signup`                                       |
| -    | Email verification         | 6-digit code                                                                          | Everyone     | `POST /auth/send-verification`, `POST /auth/verify-email` |
| 1    | Personal info              | first\_name, last\_name, d\_o\_b, gender, bh\_nationality\_country\_id, phone\_number | Everyone     | `PATCH /onboarding/user`                                  |
| 2    | Address                    | address\_1, address\_2, city, state, bh\_country\_id, post\_code                      | Everyone     | `POST /onboarding/address`                                |
| 3    | Residence country          | bh\_residence\_country\_id, language, currency\_id                                    | Everyone     | `PATCH /onboarding/user`                                  |
| 4    | Username                   | username                                                                              | Everyone     | `PATCH /onboarding/user`                                  |
| 5    | Measurements and habits    | height, weight, units, exercise, alcohol, smoking, caffeine                           | Medical only | `POST /onboarding/health`                                 |
| 6    | Body composition           | body\_type, body\_fat\_percentage                                                     | Medical only | `POST /onboarding/health`                                 |
| 7    | Conditions and medications | medical\_condition\_ids, medications array                                            | Medical only | `POST /onboarding/health`, `POST /onboarding/medication`  |

{% hint style="info" %}
Recreational users complete steps 0 through 4 and skip directly to `POST /onboarding/complete`. Medical users complete all steps through 7. Health steps are optional for recreational users but can be filled in later from profile settings.
{% endhint %}

***

## Residence country restriction

Residence country can only be changed once every 6 months. This is enforced inside `PATCH /onboarding/user` whenever `bh_residence_country_id` is present in the request.

The endpoint checks:

1. Has the user set a residence country before? (is `residence_country_changed_at` not null?)
2. If yes, have 6 months passed since the last change? (is `residence_country_changed_at + 15768000000ms <= now`?)
3. If fewer than 6 months have passed, the request is rejected with a 403 error
4. If the check passes, `residence_country_changed_at` is automatically updated to the current timestamp

This restriction applies both during onboarding and any future updates to the residence country.

***

## Endpoints

### PATCH /onboarding/user

Updates user profile fields for any onboarding step. The frontend sends only the fields relevant to the current step along with the `onboarding_step` number. Only the fields included in the request are changed.

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

Only include the fields relevant to the current step. All fields are optional.

| Field                       | Type    | Description                                         |
| --------------------------- | ------- | --------------------------------------------------- |
| `first_name`                | Text    |                                                     |
| `last_name`                 | Text    |                                                     |
| `full_name`                 | Text    |                                                     |
| `d_o_b`                     | Date    |                                                     |
| `gender`                    | Text    | `Male`, `Female`, `Non Binary`, `Prefer Not To Say` |
| `bh_nationality_country_id` | Integer | BudHub country ID                                   |
| `phone_number`              | Text    |                                                     |
| `bh_residence_country_id`   | Integer | Triggers the 6-month change restriction             |
| `username`                  | Text    |                                                     |
| `sub_type`                  | Text    | `Recreational`, `Medical`                           |
| `language`                  | Text    | Locale format, for example `en_gb`                  |
| `currency_id`               | Integer |                                                     |
| `onboarding_step`           | Integer | The step just completed                             |

**Example request (step 1: personal info):**

```json
{
  "first_name": "Leonardo",
  "last_name": "Quinteros",
  "full_name": "Leonardo Quinteros",
  "d_o_b": "1990-04-14",
  "gender": "Male",
  "bh_nationality_country_id": 209,
  "phone_number": "+34612345678",
  "onboarding_step": 1
}
```

**Example request (step 3: residence country):**

```json
{
  "bh_residence_country_id": 209,
  "language": "es_es",
  "currency_id": 26,
  "onboarding_step": 3
}
```

**Example request (step 4: username):**

```json
{
  "username": "lquint",
  "onboarding_step": 4
}
```

**Response:** Returns the updated user object.

**Errors:**

| Status | Message                                              | When                                                                      |
| ------ | ---------------------------------------------------- | ------------------------------------------------------------------------- |
| `403`  | Residence country can only be changed every 6 months | Attempting to change residence country within 6 months of the last change |

***

### POST /onboarding/address

Creates or updates the user's default address. If a default address already exists for the user, it is updated with the new data. If no default address exists, a new one is created with `default = true`.

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

| Field             | Type    | Required | Description                              |
| ----------------- | ------- | -------- | ---------------------------------------- |
| `full_name`       | Text    | Yes      | Full name of the person at this address  |
| `phone_number`    | Text    | Yes      | Contact phone number for this address    |
| `bh_country_id`   | Integer | Yes      | Country. References BudHub country table |
| `address_1`       | Text    | Yes      | Primary street address                   |
| `address_2`       | Text    | No       | Apartment, suite, floor                  |
| `city`            | Text    | Yes      | City or town                             |
| `state`           | Text    | Yes      | State, province, or region               |
| `post_code`       | Text    | Yes      | Postal or ZIP code                       |
| `onboarding_step` | Integer | No       | The step just completed                  |

**Example request:**

```json
{
  "full_name": "Leonardo Quinteros",
  "phone_number": "+34612345678",
  "bh_country_id": 209,
  "address_1": "Calle Gran Via 28",
  "address_2": "Piso 3",
  "city": "Madrid",
  "state": "Madrid",
  "post_code": "28013",
  "onboarding_step": 2
}
```

**Response:** Returns the address object (created or updated).

***

### POST /onboarding/health

Creates or updates the user's health profile. If a health record already exists for the user, it is updated with the fields provided. If no record exists, a new one is created. All fields are optional so the frontend can send data across multiple screens. Also links the health record to the user and updates the onboarding step.

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

All fields are optional. Send only the fields relevant to the current screen.

| Field                   | Type          | Description                                       |
| ----------------------- | ------------- | ------------------------------------------------- |
| `height`                | Decimal       | Height value                                      |
| `height_unit`           | Text          | `cm` or `ft`                                      |
| `weight`                | Decimal       | Weight value                                      |
| `weight_unit`           | Text          | `kg` or `lb`                                      |
| `sex_at_birth`          | Text          | `Male`, `Female`, `Intersex`, `Prefer Not To Say` |
| `exercise_frequency`    | Integer       | References `habits_frequency` table               |
| `body_type`             | Text          | `Ectomorph`, `Mesomorph`, `Endomorph`             |
| `alcohol_consumption`   | Integer       | References `habits_frequency` table               |
| `smoking_habit`         | Integer       | References `habits_frequency` table               |
| `caffeine_consumption`  | Integer       | References `habits_frequency` table               |
| `body_fat_percentage`   | Decimal       | Optional                                          |
| `medical_condition_ids` | Integer array | Array of BudHub use IDs                           |
| `onboarding_step`       | Integer       | The step just completed                           |

**Example request (step 5: measurements and habits):**

```json
{
  "height": 178,
  "height_unit": "cm",
  "weight": 75,
  "weight_unit": "kg",
  "sex_at_birth": "Male",
  "exercise_frequency": 3,
  "alcohol_consumption": 2,
  "smoking_habit": 1,
  "caffeine_consumption": 4,
  "onboarding_step": 5
}
```

**Example request (step 6: body composition):**

```json
{
  "body_type": "Mesomorph",
  "body_fat_percentage": 18.5,
  "onboarding_step": 6
}
```

**Example request (step 7: medical conditions):**

```json
{
  "medical_condition_ids": [3, 7, 12],
  "onboarding_step": 7
}
```

**Response:** Returns the health profile object (created or updated).

***

### POST /onboarding/medication

Creates multiple medication records in a single call. The frontend collects all medications locally and sends them as an array when the user taps proceed.

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

| Field             | Type             | Required | Description                |
| ----------------- | ---------------- | -------- | -------------------------- |
| `medications`     | Array of objects | Yes      | List of medication objects |
| `onboarding_step` | Integer          | No       | The step just completed    |

Each medication object contains:

| Field           | Type    | Description                                                       |
| --------------- | ------- | ----------------------------------------------------------------- |
| `bh_use_id`     | Integer | The condition this medication treats. References BudHub use table |
| `name`          | Text    | Medication name (for example, "Ibuprofen")                        |
| `dose_quantity` | Decimal | Amount per dose (for example, 400)                                |
| `dose_unit`     | Text    | `mg`, `ml`, `g`, `mcg`, `IU`                                      |
| `frequency`     | Text    | How often taken (for example, "twice daily")                      |

**Example request:**

```json
{
  "medications": [
    {
      "bh_use_id": 3,
      "name": "Ibuprofen",
      "dose_quantity": 400,
      "dose_unit": "mg",
      "frequency": "twice daily"
    },
    {
      "bh_use_id": 7,
      "name": "Sertraline",
      "dose_quantity": 50,
      "dose_unit": "mg",
      "frequency": "once daily"
    }
  ],
  "onboarding_step": 7
}
```

**Response:**

```json
{
  "message": "Medications saved successfully"
}
```

***

### POST /onboarding/complete

Marks the onboarding process as finished. Validates that all required steps have been completed before allowing completion. Sets `onboarding_complete = true` and `first_login = false`.

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

| Field             | Type    | Required | Description                                                |
| ----------------- | ------- | -------- | ---------------------------------------------------------- |
| `onboarding_step` | Integer | Yes      | The final step reached (4 for recreational, 7 for medical) |

**Example request:**

```json
{
  "onboarding_step": 4
}
```

**Response:**

```json
{
  "message": "Onboarding complete"
}
```

**Errors:**

| Status | Message                                             | When                                                 |
| ------ | --------------------------------------------------- | ---------------------------------------------------- |
| `403`  | Email must be verified before completing onboarding | `email_verified` is false                            |
| `403`  | Personal information is required                    | `first_name` is null                                 |
| `403`  | Address is required                                 | No default address exists                            |
| `403`  | Residence country is required                       | `bh_residence_country_id` is null                    |
| `403`  | Username is required                                | `username` is null                                   |
| `403`  | Health information is required for medical users    | `sub_type` is `Medical` and `user_health_id` is null |

**Backend validations:**

The endpoint checks that all mandatory steps are completed regardless of what `onboarding_step` value the frontend sends. This prevents users from bypassing the onboarding flow by calling the complete endpoint directly.

{% tabs %}
{% tab title="Recreational user" %}
Required before completion: email verified, personal info, address, residence country, username. Health steps are optional. Frontend sends `onboarding_step: 4`.
{% endtab %}

{% tab title="Medical user" %}
Required before completion: everything a recreational user needs, plus a health record. Medications are optional even for medical users. Frontend sends `onboarding_step: 7`.
{% endtab %}
{% endtabs %}

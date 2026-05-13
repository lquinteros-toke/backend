---
icon: key
---

# Authentication

Handles user account creation, login, logout, email verification, and pre-signup validation. Signup, login, and logout are processed through the Gateway, which issues a `gateway_token` alongside the app's own `authToken`. Email verification and validation endpoints are called directly on Toke.

## Endpoints through the Gateway

These endpoints are called on the Gateway, not directly on Toke. The Gateway forwards the request to Toke, creates a session, and returns both tokens.

**Gateway base URL:** `https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo`

### POST /auth/signup

Creates a new user account in Toke, generates a unique referral code, processes the referral if a valid code was provided, and returns both a Gateway JWT and a Toke authToken.

**Called on:** Gateway

**Auth:** None (public endpoint, no token required)

**Headers:**

| Header          | Value              | Required |
| --------------- | ------------------ | -------- |
| `Content-Type`  | `application/json` | Yes      |
| `X-Data-Source` | `test` or `live`   | Yes      |
| `X-Branch`      | `dev` or `v1`      | Yes      |

**Body:**

| Field           | Type  | Required | Description                                                                |
| --------------- | ----- | -------- | -------------------------------------------------------------------------- |
| `app_key`       | Text  | Yes      | Toke's app key (Key 3). Identifies which app is calling the Gateway        |
| `email`         | Email | Yes      | Must not already exist in the database                                     |
| `password`      | Text  | Yes      | Minimum 8 characters, at least 1 letter and 1 digit                        |
| `sub_type`      | Text  | No       | Cannabis preference. Values: `Recreational`, `Medical`                     |
| `referral_code` | Text  | No       | Another user's referral code. If valid, both users receive welcome credits |

**Example request:**

```
POST https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo/auth/signup

Headers:
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev

Body:
{
  "app_key": "<TOKE_APP_KEY>",
  "email": "leo@example.com",
  "password": "MyPassword1",
  "sub_type": "Recreational",
  "referral_code": "7F36SC"
}
```

What happens behind the scenes:

{% stepper %}
{% step %}
### Gateway validates app

The Gateway verifies the `app_key` against the `app_client` table and identifies the caller as Toke.
{% endstep %}

{% step %}
### Forward signup to Toke

The Gateway forwards the signup data to Toke's `/auth/signup` endpoint.
{% endstep %}

{% step %}
### Toke creates user and processes referral

Toke checks if the email already exists, creates the user, generates a unique referral code, and processes the referral if a valid code was provided (creates credits for both users).
{% endstep %}

{% step %}
### Toke returns authToken

Toke returns its own `authToken`.
{% endstep %}

{% step %}
### Gateway retrieves profile

The Gateway calls Toke's `/auth/me` using the `authToken` to retrieve the full user profile.
{% endstep %}

{% step %}
### Gateway generates gateway\_token

The Gateway generates a `gateway_token` (JWT) with `main_type: "Customer"` (hardcoded for all Toke users).
{% endstep %}

{% step %}
### Gateway saves session and returns tokens

The Gateway saves the session in the `auth_tokens` table and both tokens are returned to the frontend.
{% endstep %}
{% endstepper %}

**Response:**

```json
{
  "gateway_token": "eyJhbGciOiJIUzI1NiJ9...",
  "app_token": "eyJhbGciOiJBMjU2S1ci...",
  "expires_at": 1775903025639,
  "user_id": 3,
  "app": "toke"
}
```

| Field           | Description                                                             |
| --------------- | ----------------------------------------------------------------------- |
| `gateway_token` | JWT for cross-app calls through the Gateway. Store as `gateway_token`   |
| `app_token`     | Toke's own authToken for direct calls to Toke DB. Store as `toke_token` |
| `expires_at`    | Gateway token expiration timestamp (30 days from creation)              |
| `user_id`       | The new user's ID in Toke DB                                            |
| `app`           | Always `toke` for Toke signups                                          |

**Errors:**

| Status | Message                        | Source                         |
| ------ | ------------------------------ | ------------------------------ |
| `401`  | Unauthorized app               | Gateway (invalid app\_key)     |
| `409`  | This account is already in use | Toke (email already exists)    |
| `404`  | User not found                 | Gateway (/auth/me call failed) |

{% tabs %}
{% tab title="Without referral" %}
User is created with a unique referral code and zero credits. They can share their code with others immediately. Both tokens are returned.
{% endtab %}

{% tab title="With valid referral" %}
User is created and both the new user and the referrer receive pending credits. Credit amounts come from the `app_settings` table (configurable by admins). Both tokens are returned.
{% endtab %}

{% tab title="With invalid referral" %}
User is created normally. The invalid code is silently ignored. No error is thrown and no credits are created. Both tokens are returned.
{% endtab %}
{% endtabs %}

{% hint style="info" %}
The referral processing (credit creation, user updates) happens entirely inside Toke's backend. The Gateway simply forwards the `referral_code` field and Toke handles the rest.
{% endhint %}

{% hint style="warning" %}
The `main_type` in the Gateway JWT is never taken from the frontend input. For Toke users it is always hardcoded as `Customer`. This prevents a user from forging their access level.
{% endhint %}

### POST /auth/login

Authenticates an existing Toke user and returns both a Gateway JWT and a Toke authToken.

**Called on:** Gateway

**Auth:** None (public endpoint, no token required)

**Headers:**

| Header          | Value              | Required |
| --------------- | ------------------ | -------- |
| `Content-Type`  | `application/json` | Yes      |
| `X-Data-Source` | `test` or `live`   | Yes      |
| `X-Branch`      | `dev` or `v1`      | Yes      |

**Body:**

| Field      | Type  | Required | Description              |
| ---------- | ----- | -------- | ------------------------ |
| `app_key`  | Text  | Yes      | Toke's app key (Key 3)   |
| `email`    | Email | Yes      | The user's email address |
| `password` | Text  | Yes      | The user's password      |

**Example request:**

```
POST https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo/auth/login

Headers:
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev

Body:
{
  "app_key": "<TOKE_APP_KEY>",
  "email": "leo@example.com",
  "password": "MyPassword1"
}
```

What happens behind the scenes:

{% stepper %}
{% step %}
### Gateway validates app

The Gateway verifies the `app_key` and identifies the caller as Toke.
{% endstep %}

{% step %}
### Forward credentials to Toke

The Gateway forwards email and password to Toke's `/auth/login` endpoint.
{% endstep %}

{% step %}
### Toke validates credentials

Toke validates the credentials and returns its own `authToken`. If invalid, returns `401` which the Gateway forwards to the frontend.
{% endstep %}

{% step %}
### Gateway retrieves profile

The Gateway calls Toke's `/auth/me` using the `authToken` to retrieve the user profile.
{% endstep %}

{% step %}
### Gateway generates gateway\_token

The Gateway generates a `gateway_token` with `main_type: "Customer"`.
{% endstep %}

{% step %}
### Gateway saves session and returns tokens

The Gateway saves the session in the `auth_tokens` table and both tokens are returned to the frontend.
{% endstep %}
{% endstepper %}

**Response:**

```json
{
  "gateway_token": "eyJhbGciOiJIUzI1NiJ9...",
  "app_token": "eyJhbGciOiJBMjU2S1ci...",
  "expires_at": 1775903025639,
  "user_id": 1,
  "app": "toke"
}
```

**Errors:**

| Status | Message             | Source                         |
| ------ | ------------------- | ------------------------------ |
| `401`  | Unauthorized app    | Gateway (invalid app\_key)     |
| `401`  | Invalid Credentials | Toke (wrong email or password) |
| `404`  | User not found      | Gateway (/auth/me call failed) |

### POST /auth/logout

Revokes the Gateway session. The `gateway_token` becomes permanently invalid.

**Called on:** Gateway

**Auth:** Requires `gateway_token`

**Headers:**

| Header          | Value                    | Required |
| --------------- | ------------------------ | -------- |
| `Authorization` | `Bearer <gateway_token>` | Yes      |
| `Content-Type`  | `application/json`       | Yes      |
| `X-Data-Source` | `test` or `live`         | Yes      |
| `X-Branch`      | `dev` or `v1`            | Yes      |

**Body:** None

**Example request:**

```
POST https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo/auth/logout

Headers:
  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev
```

What happens behind the scenes:

{% stepper %}
{% step %}
### Validate token

The Gateway decodes and validates the `gateway_token`.
{% endstep %}

{% step %}
### Revoke session

The Gateway sets `is_revoked = true` on the corresponding `auth_tokens` record.
{% endstep %}

{% step %}
### Return success

Returns a success message.
{% endstep %}
{% endstepper %}

**Response:**

```json
{
  "message": "Logged out successfully"
}
```

What the frontend must do after receiving this response:

1. Delete `gateway_token` from Secure Storage
2. Delete `toke_token` from Secure Storage
3. Redirect to the login screen

{% hint style="warning" %}
The revoked `gateway_token` cannot be reused. Any subsequent call with it will be rejected. The `toke_token` is still technically valid for direct Toke calls, which is why the frontend must delete it locally. For full security, consider also calling Toke's own logout endpoint if one exists.
{% endhint %}

## Token storage after login or signup

{% tabs %}
{% tab title="Flutter (Toke)" %}
| Token           | Key in Secure Storage | Used for                                                  |
| --------------- | --------------------- | --------------------------------------------------------- |
| `gateway_token` | `gateway_token`       | Cross-app calls to BudHub data through the Gateway        |
| `app_token`     | `toke_token`          | Direct calls to Toke DB (profile, reviews, baskets, etc.) |
{% endtab %}
{% endtabs %}

## Endpoints called directly on Toke

These endpoints are called directly on Toke using the `toke_token`. They do not go through the Gateway.

**Toke Authentication base URL:** `https://xads-j1s5-11eg.f2.xano.io/api:MxyG7FML`

### GET /auth/me

Returns the authenticated user's basic profile information.

**Called on:** Toke directly

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:** None

**Example request:**

```
GET https://xads-j1s5-11eg.f2.xano.io/api:MxyG7FML/auth/me

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev
```

**Response:**

```json
{
  "id": 1,
  "created_at": 1773305859056,
  "name": "Leo",
  "email": "leo@example.com",
  "main_type": "Customer"
}
```

### POST /auth/send-verification

Generates a 6-digit verification code and sends it to the user's email via SendGrid. Codes expire after 10 minutes. A 60-second cooldown prevents repeated requests.

**Called on:** Toke directly

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:** None

**Example request:**

```
POST https://xads-j1s5-11eg.f2.xano.io/api:MxyG7FML/auth/send-verification

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev
```

**Response:**

```json
{
  "message": "Verification code sent",
  "email": "leo@example.com"
}
```

**Errors:**

| Status | Message                                  | When                                              |
| ------ | ---------------------------------------- | ------------------------------------------------- |
| `400`  | Email is already verified                | User has already verified their email             |
| `429`  | Please wait before requesting a new code | Less than 60 seconds since the last code was sent |

{% hint style="info" %}
The 60-second rate limit is enforced by checking `verification_code_expires_at` on the user record. Codes expire after 10 minutes.
{% endhint %}

### POST /auth/verify-email

Validates the 6-digit code entered by the user. If correct and not expired, marks the email as verified and clears the code from the database.

**Called on:** Toke directly

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Body:**

| Field  | Type | Required | Description                         |
| ------ | ---- | -------- | ----------------------------------- |
| `code` | Text | Yes      | The 6-digit code received via email |

**Example request:**

```
POST https://xads-j1s5-11eg.f2.xano.io/api:MxyG7FML/auth/verify-email

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev

Body:
{
  "code": "483921"
}
```

**Response:**

```json
{
  "message": "Email verified successfully"
}
```

**Errors:**

| Status | Message                                                 | When                              |
| ------ | ------------------------------------------------------- | --------------------------------- |
| `400`  | Email is already verified                               | Already verified                  |
| `400`  | No verification code has been requested                 | No code exists on the user record |
| `400`  | Invalid verification code                               | Code does not match               |
| `410`  | Verification code has expired. Please request a new one | More than 10 minutes have passed  |

### GET /auth/check-referral/{referral\_code}

Checks if a referral code belongs to an existing user. Used by the frontend to validate the code in real time before the user submits the signup form.

**Called on:** Toke directly

**Auth:** None (public endpoint)

**Headers:**

| Header          | Value              | Required |
| --------------- | ------------------ | -------- |
| `Content-Type`  | `application/json` | Yes      |
| `X-Data-Source` | `test` or `live`   | Yes      |
| `X-Branch`      | `dev` or `v1`      | Yes      |

**Path parameters:**

| Parameter       | Type | Description          |
| --------------- | ---- | -------------------- |
| `referral_code` | Text | The code to validate |

**Example request:**

```
GET https://xads-j1s5-11eg.f2.xano.io/api:MxyG7FML/auth/check-referral/7F36SC

Headers:
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev
```

**Response (valid):**

```json
{
  "valid": true,
  "referrer_name": "Leo"
}
```

**Response (invalid):**

```json
{
  "valid": false
}
```

{% hint style="info" %}
The response is only within the backend calls, so no data is exposed.
{% endhint %}

### GET /auth/check-username/{username}

Checks if a username is already taken. Used for real-time validation as the user types during onboarding or profile editing.

**Called on:** Toke directly

**Auth:** Xano built-in (requires `toke_token`)

**Headers:**

| Header          | Value                 | Required |
| --------------- | --------------------- | -------- |
| `Authorization` | `Bearer <toke_token>` | Yes      |
| `Content-Type`  | `application/json`    | Yes      |
| `X-Data-Source` | `test` or `live`      | Yes      |
| `X-Branch`      | `dev` or `v1`         | Yes      |

**Path parameters:**

| Parameter  | Type | Description           |
| ---------- | ---- | --------------------- |
| `username` | Text | The username to check |

**Example request:**

```
GET https://xads-j1s5-11eg.f2.xano.io/api:MxyG7FML/auth/check-username/lquint

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev
```

**Response (available):**

```json
{
  "available": true
}
```

**Response (taken):**

```json
{
  "available": false
}
```

## Functions

### generate\_referral\_code

Generates a unique 6-character uppercase alphanumeric code. Called during signup inside Toke's backend to assign every new user their own referral code.

**Inputs:** None

**How it works:** Generates a random 6-character string, checks the `user` table for duplicates, and regenerates if a collision is found. Only used inside the backend.

**Returns:** A unique text string (for example, `7F36SC`)

### process\_referral

Validates a referral code and creates welcome credits for both users if valid. Called during signup inside Toke's backend only when the new user provides a referral code.

**Inputs:**

| Input           | Type    | Description                      |
| --------------- | ------- | -------------------------------- |
| `referral_code` | Text    | The code entered by the new user |
| `new_user_id`   | Integer | The ID of the newly created user |

How it works:

{% stepper %}
{% step %}
### Lookup code

Looks up the referral code in the `user` table.
{% endstep %}

{% step %}
### No match

If no match is found, returns `{ valid: false }` and the signup proceeds without credits.
{% endstep %}

{% step %}
### Match found — read settings

If a match is found, reads credit amounts from the `app_settings` table.
{% endstep %}

{% step %}
### Create credits for new user

Creates a `user_credit` record for the new user with source `Welcome Credit` and status `Pending`.
{% endstep %}

{% step %}
### Create credits for referrer

Creates a `user_credit` record for the referrer with source `Friend Referred` and status `Pending`.
{% endstep %}

{% step %}
### Link credits

Links both credit records together via `counterpart_credit_id`.
{% endstep %}

{% step %}
### Update user records

Updates both users: adds the credit amount to `credit_pending` and `credit_total`, sets `used_referral_code` on the new user.
{% endstep %}
{% endstepper %}

**Returns:** `{ valid: true, referrer_id: <id> }` or `{ valid: false }`

{% hint style="info" %}
Most of credits are created with status `Pending`. They become spendable only after their status is updated to `Received` through a separate process, such as after the referred user's first purchase or a configurable waiting period.
{% endhint %}



## Endpoint summary

| Endpoint                              | Called on | Auth                    | Purpose                                  |
| ------------------------------------- | --------- | ----------------------- | ---------------------------------------- |
| `POST /auth/signup`                   | Gateway   | None (app\_key in body) | Create account and get both tokens       |
| `POST /auth/login`                    | Gateway   | None (app\_key in body) | Authenticate and get both tokens         |
| `POST /auth/logout`                   | Gateway   | `gateway_token`         | Revoke the Gateway session               |
| `GET /auth/me`                        | Toke      | `toke_token`            | Get current user profile                 |
| `POST /auth/send-verification`        | Toke      | `toke_token`            | Send 6-digit verification code via email |
| `POST /auth/verify-email`             | Toke      | `toke_token`            | Validate code and mark email verified    |
| `GET /auth/check-referral/{code}`     | Toke      | None                    | Check if a referral code is valid        |
| `GET /auth/check-username/{username}` | Toke      | `toke_token`            | Check if a username is available         |

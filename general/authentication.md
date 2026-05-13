---
icon: key
---

# Authentication

How authentication and token issuance works for each auth flow. The Gateway is the single authority that issues `gateway_tokens` and `app_tokens`

Each flow below shows exactly what is sent, what is checked, and what is returned.

BASE\_URL: [`https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo/`](https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo/)

***

## Signup

A new user creates an account. .

**Endpoint:** `POST /signup`\
No `Authorization` header required&#x20;

### What the frontend sends

{% tabs %}
{% tab title="Headers" %}
| Header          | Value              |
| --------------- | ------------------ |
| `Content-Type`  | `application/json` |
| `X-Data-Source` | `test` or `live`   |
| `X-Branch`      | `dev` or `v1`      |
{% endtab %}

{% tab title="Body" %}
| Field            | Value                             |
| ---------------- | --------------------------------- |
| `user_email`     | `user@example.com`                |
| `password`       | `••••••`                          |
| `user_main_type` | `Admin`, `Business` or `Customer` |
| `referral_code`  | `ABC123`                          |
| `branch`         | `dev` or `v1`                     |
{% endtab %}
{% endtabs %}

### What the Gateway does

{% stepper %}
{% step %}
#### Verify the app

Checks `app_key` against the `app_client` table → identifies which app (Toke or BudHub)
{% endstep %}

{% step %}
#### Forward to app

Sends email, password, and profile fields to the app's own `auth/signup` endpoint
{% endstep %}

{% step %}
#### App creates user

The app creates the user in its database and returns its own `authToken`
{% endstep %}

{% step %}
#### Fetch user object

Gateway calls `/auth/me` with the `authToken` to get the full user object including `main_type`
{% endstep %}

{% step %}
#### Generate gateway\_token

Creates a signed JWT with: `user_id`, `email`, `app_client_id`, `main_type`
{% endstep %}

{% step %}
#### Save session

Stores the `gateway_token` and `app_token` in the `auth_tokens` table
{% endstep %}
{% endstepper %}

### How main\_type is determined

{% tabs %}
{% tab title="Toke user" %}
`main_type` = **Customer**

Always hardcoded. Every Toke user is a Customer. The frontend's `sub_type` field (`recreational` or `medical`) is stored in the Toke user table but does not affect the main access level.
{% endtab %}

{% tab title="BudHub user" %}
`main_type` = **Admin** or **Business**

Read from the `/auth/me` response — never from the signup body. Even if a frontend sends `main_type: "Admin"` in the signup body, the Gateway ignores it. The JWT's `main_type` is always sourced from the database.
{% endtab %}
{% endtabs %}

{% hint style="danger" %}
**Security:** A user cannot forge their access level. Even if `main_type: "Admin"` is sent in the signup body, the Gateway reads `main_type` from the database via `/me` after the user is created. The signup body value is only used by the app to store in its own user table.
{% endhint %}

### What the frontend receives

```json
{
  "gateway_token": "eyJhbGciOiJIUzI1NiJ9...",
  "app_token": "eyJhbGciOiJBMjU2S1ci...",
  "expires_at": 1775903025639,
  "user_id": 3,
  "app": "toke"
}
```

| Field           | Description                            |
| --------------- | -------------------------------------- |
| `gateway_token` | JWT for cross-app calls via Gateway    |
| `app_token`     | Xano authToken for own DB direct calls |
| `expires_at`    | Token expiration timestamp             |
| `user_id`       | The new user's ID                      |
| `app`           | `"toke"` or `"budhub"`                 |

{% hint style="info" %}
The user is logged in immediately after signup. The frontend stores both tokens right away.
{% endhint %}

### Error handling

{% tabs %}
{% tab title="User already exists" %}
The app returns `401` → Gateway forwards the error directly to the frontend. No token is created.
{% endtab %}
{% endtabs %}

***

## Login

An existing user authenticates. Same flow as signup, but the app validates credentials instead of creating a user.

**Endpoint:** `POST /login`\
No `Authorization` header required — the user is authenticating to get tokens.

### What the frontend sends

{% tabs %}
{% tab title="Headers" %}
| Header          | Value              |
| --------------- | ------------------ |
| `Content-Type`  | `application/json` |
| `X-Data-Source` | `test` or `live`   |
| `X-Branch`      | `dev` or `v1`      |
{% endtab %}

{% tab title="Body" %}
| Field      | Value               |
| ---------- | ------------------- |
| `app_key`  | Toke or Budhub keys |
| `email`    | `user@example.com`  |
| `password` | `••••••`            |
| `branch`   | `test` or `live`    |
{% endtab %}
{% endtabs %}

### What the Gateway does

{% stepper %}
{% step %}
#### Verify the app

Checks `app_key` → identifies which app
{% endstep %}

{% step %}
#### Forward credentials

Sends email + password to the app's `/auth/login`
{% endstep %}

{% step %}
#### App validates

App checks credentials against its DB → returns `authToken`. If invalid → `401` forwarded to frontend
{% endstep %}

{% step %}
#### Fetch user object

Gateway calls `/auth/me` with `authToken` → gets user object + `main_type`
{% endstep %}

{% step %}
#### Generate gateway\_token

Creates JWT with `user_id`, `email`, `app_client_id`, `main_type`
{% endstep %}

{% step %}
#### Save session

Stores in `auth_tokens` table
{% endstep %}
{% endstepper %}

### Error handling

{% tabs %}
{% tab title="Invalid credentials" %}
The app returns `401` → Gateway forwards the error directly to the frontend. No token is created.
{% endtab %}

{% tab title="Invalid app_key" %}
`verify_app_client` fails → Gateway returns `401 "Unauthorized app"`. The request never reaches the app's database.
{% endtab %}

{% tab title="Email not found" %}
The app returns `404` or `401` → Gateway forwards the error. No token is created.
{% endtab %}
{% endtabs %}

### What the frontend receives

Same response structure as signup:

```json
{
  "gateway_token": "eyJhbGciOiJIUzI1NiJ9...",
  "app_token": "eyJhbGciOiJBMjU2S1ci...",
  "expires_at": 1775903025639,
  "user_id": 1,
  "app": "toke"
}
```

### Token storage

{% tabs %}
{% tab title="Flutter (Toke)" %}
| Token                               | Storage        | Used for                              |
| ----------------------------------- | -------------- | ------------------------------------- |
| `gateway_token`                     | Secure Storage | Cross-app calls to BudHub via Gateway |
| `app_token` → saved as `toke_token` | Secure Storage | Direct calls to Toke DB               |
{% endtab %}

{% tab title="Bubble (BudHub)" %}
| Token                                 | Storage           | Used for                            |
| ------------------------------------- | ----------------- | ----------------------------------- |
| `gateway_token`                       | Bubble User table | Cross-app calls to Toke via Gateway |
| `app_token` → saved as `budhub_token` | Bubble User table | Direct calls to BudHub DB           |
{% endtab %}
{% endtabs %}

***

## Logout

The user ends their session. The `gateway_token` is revoked server-side and deleted locally.

**Endpoint:** `POST /logout`\
Requires the `gateway_token`

### What the frontend sends

| Header          | Value                    |
| --------------- | ------------------------ |
| `Authorization` | `Bearer <gateway_token>` |
| `Content-Type`  | `application/json`       |
| `X-Data-Source` | `test` or `live`         |

No body needed

### What the Gateway does

{% stepper %}
{% step %}
#### Validate the token

Calls `validate_jwt` to decode the token and confirm it's valid and not already revoked
{% endstep %}

{% step %}
#### Revoke the session

Updates the `auth_tokens` record: sets `is_revoked = true`
{% endstep %}

{% step %}
#### Return confirmation

Sends success message back to the frontend
{% endstep %}
{% endstepper %}

### What the frontend receives

```json
{
  "message": "Logged out successfully"
}
```

### What the frontend does after

{% tabs %}
{% tab title="Flutter (Toke)" %}
1. Delete `gateway_token` from Secure Storage
2. Delete `toke_token` from Secure Storage
3. Redirect to login screen
{% endtab %}

{% tab title="Bubble (BudHub)" %}
1. Clear `gateway_token` from User table
2. Clear `budhub_token` from User table
3. Logs the user out from Bubble
4. Redirect to login screen
{% endtab %}
{% endtabs %}

{% hint style="warning" %}
**After logout:** The revoked `gateway_token` cannot be reused — any subsequent call with it will be rejected by `validate_jwt` because `is_revoked = true`. The `app_token` (`toke_token` or `budhub_token`) is still technically valid on its own app's DB, which is why the frontend must delete it locally. For full security, the app should also have its own logout endpoint to invalidate the `app_token`.
{% endhint %}

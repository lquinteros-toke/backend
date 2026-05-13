---
icon: lock-open
---

# Password Reset

How users request a one-time code to reset their password. The flow generates a short-lived numeric code, emails it to the user, verifies it, and exchanges it for an auth token used to set a new password.

## Overview

The OTP password reset flow sends a 6-character code directly to the user's email. The code is uppercase letters and digits only, making it easy to type on mobile without ambiguity. It is stored on the user record alongside an expiration timestamp and is valid for 15 minutes. When the user submits the correct code, the endpoint validates it and immediately issues a standard Xano auth token, which is then used to authenticate the password update call. The code is cleared from the user record at that point and cannot be reused.

## Flow Overview

```
Step 1: User requests a reset code
         Flutter → Toke POST /reset-code/request
                    │
                    ├─ Validates email exists
                    ├─ Generates 6-character code (uppercase + digits)
                    ├─ Stores code + expiry on user record
                    ├─ Sends code via SendGrid plain text email
                    │
                    ▼
         User receives email with code

Step 2: User submits the code
         Flutter → Toke POST /reset-code/check
                    │
                    ├─ Validates email exists
                    ├─ Compares submitted code against stored code
                    ├─ Checks code has not expired
                    ├─ Issues Xano auth token (24h)
                    ├─ Clears reset_code + reset_code_expires_at
                    │
                    ▼
         Returns authToken → Flutter stores it

Step 3: User sets new password
         Flutter → Toke POST /update_password
                    │
                    ├─ Auth: uses the authToken from Step 2
                    ├─ Updates password on user record
                    │
                    ▼
         Password changed
```

## Endpoints

### GET /reset-code/request

Generates a 6-character reset code and emails it to the user. The code is stored on the user record with a 15-minute expiry. Any previously issued code is overwritten.

**Auth:** None (public endpoint, no token required)

**Headers:**

| Header          | Value              | Required |
| --------------- | ------------------ | -------- |
| `Content-Type`  | `application/json` | Yes      |
| `X-Data-Source` | `test` or `live`   | Yes      |
| `X-Branch`      | `dev` or `v1`      | Yes      |

**Body:**

| Field   | Type  | Required | Description              |
| ------- | ----- | -------- | ------------------------ |
| `email` | Email | Yes      | The user's email address |

**Example request:**

```http
POST https://xads-j1s5-11eg.f2.xano.io/api:6hazS0TY/reset-code/request

Headers:
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: v1

Body:
{
  "email": "user@example.com"
}
```

**Response:**

```json
{
  "message": "Verification code sent",
  "email": "user@example.com"
}
```

**Errors:**

| Status | Message                | When                                       |
| ------ | ---------------------- | ------------------------------------------ |
| `404`  | Email not found        | No user record matches the submitted email |
| `500`  | SendGrid error message | Email sending failed                       |

{% hint style="info" %}
This is a public endpoint — no auth token required. The email itself acts as the authentication factor (only the account owner has access to their inbox).
{% endhint %}

***

### POST /reset-code/check

Verifies the submitted OTP code against the one stored on the user record. If the code matches and has not expired, the code is cleared from the user record and a standard Xano auth token is issued. Flutter uses that token to authenticate the password update call.

**Auth:** None

**Headers:**

| Header          | Value              | Required |
| --------------- | ------------------ | -------- |
| `Content-Type`  | `application/json` | Yes      |
| `X-Data-Source` | `test` or `live`   | Yes      |
| `X-Branch`      | `dev` or `v1`      | Yes      |

**Body:**

| Field   | Type | Required | Description                                   |
| ------- | ---- | -------- | --------------------------------------------- |
| `email` | Text | Yes      | The email address associated with the account |
| `otp`   | Text | Yes      | The 6-character code received by email        |

**Example request:**

```http
POST https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo/reset-code/check

Headers:
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: v1

Body:
{
  "email": "user@example.com",
  "otp": "A3K7BX"
}
```



**Response:**

```json
{
  "authToken": "eyJhbGciOiJBMjU2S1ci..."
}
```

| Field       | Description                                                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `authToken` | Standard Xano auth token for the `user` table. Valid for 24 hours. Use this in the Authorization header for the password update call. |

**Errors:**

| Status | Message         | When                                          |
| ------ | --------------- | --------------------------------------------- |
| `500`  | Email not found | No user record matches the submitted email    |
| `500`  | Wrong code      | Submitted code does not match the stored code |
| `500`  | Expired code    | Code exists but has expired                   |

***

### POST /update\_password

Sets a new password for the authenticated user. Requires a valid auth token (obtained from check-code endpoint).

**Auth:** Xano built-in (requires `toke_token` or the `authToken` from magic login)

**Headers:**

| Header          | Value                | Required |
| --------------- | -------------------- | -------- |
| `Authorization` | `Bearer <authToken>` | Yes      |
| `Content-Type`  | `application/json`   | Yes      |
| `X-Data-Source` | `test` or `live`     | Yes      |
| `X-Branch`      | `dev` or `v1`        | Yes      |

**Body:**

| Field      | Type     | Required | Description                                              |
| ---------- | -------- | -------- | -------------------------------------------------------- |
| `password` | Password | Yes      | The new password. Hashed automatically by Xano on write. |

**Example request:**

```http
POST https://xads-j1s5-11eg.f2.xano.io/api:PhZe36yo/update_password

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev

Body:
{
  "password": "NewSecurePassword123"
}
```



**Response:**

```json
{
  "message": {
    "success": "true",
    "message": "Password updated"
  }
}
```

**Errors:**

| Status | Message       | When                                    |
| ------ | ------------- | --------------------------------------- |
| `401`  | Access Denied | Invalid, expired, or missing auth token |

{% hint style="info" %}
Password confirmation (matching passwords) is handled in the Flutter frontend before calling this endpoint. Only one password field is sent.
{% endhint %}

{% hint style="info" %}
The `password` input type ensures the value is treated as sensitive and hidden in Xano logs. Hashing happens automatically when writing to a password-type database field.
{% endhint %}

***

## Functions

### sendgrid\_dynamic\_send

Sends an email via SendGrid's API. Used to deliver the magic link to the user.

**Inputs:**

| Input      | Type  | Description                                              |
| ---------- | ----- | -------------------------------------------------------- |
| `to_email` | Email | The recipient's email address                            |
| `data`     | JSON  | Dynamic data. Must contain `magic_link` key with the URL |

**How it works:**

{% stepper %}
{% step %}
#### Validate env vars

Checks that `SENDGRID_API_KEY` and `SENDGRID_FROM_EMAIL` are set.
{% endstep %}

{% step %}
#### Call SendGrid API

Sends a POST to `https://api.sendgrid.com/v3/mail/send` with the from address, recipient, subject, and body containing the magic link.
{% endstep %}

{% step %}
#### Validate response

Checks that SendGrid returned status 202 (accepted). If not, throws the error message from SendGrid.
{% endstep %}
{% endstepper %}

**Returns:** The full API response from SendGrid.

{% hint style="info" %}
The current implementation sends plain text with the link. To use a SendGrid dynamic template, replace the `content` block in the API request params with `template_id` and move the data into `dynamic_template_data` inside `personalizations`.
{% endhint %}

***

## Environment Variables

| Variable              | Description                                  |
| --------------------- | -------------------------------------------- |
| `SENDGRID_API_KEY`    | SendGrid API key for sending emails          |
| `SENDGRID_FROM_EMAIL` | The "from" email address for outgoing emails |

---
icon: diagram-nested
---

# Endpoints

Toke's API is organized into 8 groups, each handling a specific area of the application. All endpoints are hosted on Xano and follow consistent patterns for authentication, request format, and error handling.

***

## API groups

| Group                               | api\_group\_id | Base path                                | Description                                              |
| ----------------------------------- | -------------- | ---------------------------------------- | -------------------------------------------------------- |
| [Authentication](authentication.md) | MxyG7FML       | `/auth`                                  | Signup, login, and session management                    |
| Users                               | EXX1OJAQ       | `/user`                                  | User accounts, profiles, and moderation                  |
| [Onboarding](onboarding.md)         | opDK-bNN       | `/onboarding`                            | Handles the onboarding process                           |
| Reviews                             | X4YLDfCF       | `/review`, `/review_report`              | Product and dispensary reviews, comments, votes, reports |
| Subscriptions                       | ewOU18cD       | `/user_subscription`                     | User subscription management                             |
| [Credits](credits.md)               | r5zOfm\_5      | `/user_credit`                           | Credit transactions and balance management               |
| Baskets                             | TijoL6y3       | `/basket`, `/basket_item`                | Shopping baskets and line items                          |
| Health                              | zqnm44C2       | `/user_health`, `/user_medication`       | Health profiles and medications                          |
| Addresses                           | QF1GFoad       | `/address`                               | Delivery and billing addresses                           |
| [Business](business.md)             | rOS4jyzC       | `/business`, `/business-sync`, `/nearby` | Business snapshots                                       |
| [Password reset](password-reset.md) | 6hazS0TY       | `/reset-code`, `/update_password`        | Send OTP and reset password                              |
| [Strains](strains.md)               | oGtZdRL8       | /budhub                                  | Get strains filtered                                     |
| [Products](products.md)             | oGtZdRL8       | /budhub                                  | Get products filtered                                    |

***

## Base URL

All Toke endpoints are accessed through the following base URL. The API group identifier is appended after the base domain.

```
https://xads-j1s5-11eg.f2.xano.io/api:{api_group_id}/{path}
```

Each API group has its own unique identifier. The full URL for any endpoint is the base URL combined with the group ID and the endpoint path.

***

## Required headers

Every request to Toke must include these headers. The specific token in the Authorization header depends on whether the call is made directly from the frontend or through the Gateway.

| Header          | Value              | Notes                                                            |
| --------------- | ------------------ | ---------------------------------------------------------------- |
| `Authorization` | `Bearer <token>`   | `toke_token` for direct calls, `gateway_token` for Gateway calls |
| `Content-Type`  | `application/json` | Required on all requests                                         |
| `X-Data-Source` | `test` or `live`   | Determines which data environment to use                         |
| `X-Branch`      | `dev` or `v1`      | Targets a specific API version branch. dev or v1                 |

{% hint style="info" %}
The Authentication group (`/auth/login`, `/auth/signup`) does not require an Authorization header since the user is authenticating to obtain a token. The `/auth/me` endpoint does require a valid `toke_token`.
{% endhint %}

***

## Authentication and access control

Toke uses two middlewares to protect its endpoints. The middleware determines who can access each endpoint and is documented on each endpoint's detail page.

| Middleware              | Who can access                                         | Applied to                                          |
| ----------------------- | ------------------------------------------------------ | --------------------------------------------------- |
| `auth_middleware`       | Any authenticated caller (direct user or Gateway)      | Read operations and non-sensitive actions           |
| `auth_middleware_admin` | Admin users only (via Gateway with `main_type: Admin`) | Write operations, moderation, and sensitive actions |

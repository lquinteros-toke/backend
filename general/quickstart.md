---
icon: sitemap
metaLinks:
  alternates:
    - /broken/spaces/yE16Xb3IemPxJWydtPOj/pages/7FvWQMF0kTK7HGhlQfmo
---

# Ecosystem Overview

This ecosystem consists of **three independent apps** that share data through a gateway.\
Each app has its own database and can share information with other apps, but all data relies on a clearly defined Source of Truth (SoT) to prevent duplication and inconsistencies.

<table><thead><tr><th>App</th><th>Backend</th><th>Frontend</th><th>User types</th><th width="220.6666259765625">Purpose</th></tr></thead><tbody><tr><td>BudHub</td><td>Xano</td><td>Bubble</td><td><ul><li>Admin</li><li>Business</li></ul></td><td>Business management: products, strains, orders, dispensaries, brands</td></tr><tr><td>Toke</td><td>Xano</td><td>Flutter</td><td><ul><li>Customer</li></ul></td><td>Consumer app: reviews, baskets, subscriptions, health, credits</td></tr><tr><td>Kushpoint</td><td>Xano</td><td>TBD</td><td>TBD</td><td>Dispensary POS</td></tr><tr><td>Gateway</td><td>Xano</td><td>-</td><td>-</td><td>Auth orchestration + cross-app proxy</td></tr></tbody></table>

**BudHub** is the platform where businesses (Brands and Dispensaries) can create an account, manage their own products, variants, batches, stock and orders. BudHub admins can also manage businesses, strains, strain variants and compounds.

**Toke** is the customer-facing marketplace. Here users can view and search different products, create their baskets, place and order and leave reviews.

**KushPoint** is point-of-sale software for physical dispensaries.

{% hint style="info" %}
Due to the nature of this ecosystem, there may be limited exceptions where an app is allowed to modify data owned by another app, under clearly defined rules and responsibilities.
{% endhint %}

***

## **Connection Principle**

Each app accesses its own database directly using its own Xano authToken. When an app needs data from another app's database, it goes through the Gateway.&#x20;

The Gateway verifies the caller's identity, determines their access level, and forwards the request with a service key.

Bubble (BudHub frontend) never calls Toke directly. Flutter (Toke frontend) never calls BudHub directly. All cross-app communication passes through the Gateway.

<table><thead><tr><th valign="top">Caller</th><th valign="top">Destination</th><th valign="top">Path</th><th valign="top">Auth method</th></tr></thead><tbody><tr><td valign="top">Flutter (Toke)</td><td valign="top">Toke DB</td><td valign="top">Direct</td><td valign="top">toke_token (Xano auth)</td></tr><tr><td valign="top">Flutter (Toke)</td><td valign="top">BudHub DB</td><td valign="top">Gateway proxy</td><td valign="top">gateway_token</td></tr><tr><td valign="top">Bubble (BudHub)</td><td valign="top">BudHub DB</td><td valign="top">Direct</td><td valign="top">budhub_token (Xano auth)</td></tr><tr><td valign="top">Bubble (BudHub)</td><td valign="top">Toke DB</td><td valign="top">Gateway proxy</td><td valign="top">gateway_token</td></tr></tbody></table>

***

## Tokens

When a user is created, 2 tokens will be issued and stored in the corresponding place for each app.

<table><thead><tr><th valign="top">Token</th><th valign="top">Issued by</th><th valign="top">Stored in</th><th valign="top">Used for</th></tr></thead><tbody><tr><td valign="top">gateway_token</td><td valign="top">Gateway (generate_jwt)</td><td valign="top">Flutter Secure Storage / Bubble</td><td valign="top">Cross-app calls through Gateway</td></tr><tr><td valign="top">toke_token (app_token)</td><td valign="top">Toke (Xano built-in)</td><td valign="top">Flutter Secure Storage</td><td valign="top">Direct Toke DB access from Flutter</td></tr><tr><td valign="top">budhub_token (app_token)</td><td valign="top">BudHub (Xano built-in)</td><td valign="top">Bubble</td><td valign="top">Direct BudHub DB access from Bubble</td></tr></tbody></table>

***

## **Headers**

These four headers must be present on every API request, regardless of the destination.

<table><thead><tr><th width="249">Key</th><th width="179.66668701171875">Value</th><th>Purpose</th></tr></thead><tbody><tr><td>Authorization</td><td>Bearer &#x3C;token></td><td><ul><li>gateway_token for cross-app calls</li><li>app_token for own DB calls</li></ul></td></tr><tr><td>Content-Type</td><td>application/json</td><td>Always JSON</td></tr><tr><td>X-Data-Source</td><td>test or live</td><td>Which data environment to use</td></tr><tr><td>X-Branch</td><td>dev or v1</td><td>Which API version branch to target in the first call</td></tr></tbody></table>

***

## Requests

The request structure depends on the method, the source, and the destination.

### GET requests

{% tabs %}
{% tab title="Own database (direct)" %}
Flutter → Toke DB // Bubble → BudHub DB

**Headers**\
Authorization: \<app\_token>

\
**Params**\
Send as query parameters defined by the endpoint (e.g. page, per\_page, filters, ids, etc)
{% endtab %}

{% tab title="External database (via Gateway)" %}
Flutter → Gateway → BudHub DB // Bubble → Gateway → Toke DB

**Headers**\
Authorization: \<gateway\_token>

\
**Params**\
url: Full endpoint URL of the target DB\
branch: Branch to use on the target DB\
query\_params: object that contains key-value parameters


{% endtab %}
{% endtabs %}

## POST/PATCH/DELETE requests

{% tabs %}
{% tab title="Own database (direct)" %}
Flutter → Toke DB // Bubble → BudHub DB

**Headers**\
Authorization: \<app\_token>

\
**Params**\
Send as query parameters defined by the endpoint (e.g. page, per\_page, filters, ids, etc)
{% endtab %}

{% tab title="External database (via Gateway)" %}
Flutter → Gateway → BudHub DB // Bubble → Gateway → Toke DB

**Headers**\
Authorization: \<gateway\_token>

\
**Params**\
url: Full endpoint URL of the target DB\
branch: Branch to use on the target DB\
body\_params: object that contains key-value parameters


{% endtab %}
{% endtabs %}

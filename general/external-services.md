---
icon: up-right-from-square
---

# External Services

Reference page for all third-party services used across the app. Keys and secrets are stored separately in a secure vault — this page only documents what each service is, where to manage it, and which credentials exist.

## Google Translate / Geocoding

**Purpose:** Translate data on the frontend.

**Console:** https://console.cloud.google.com

| Credential | Environment |
| ---------- | ----------- |
| API Key    | Single      |

***

## Mapbox

**Purpose:** Display dispensary locations on a map.

**Console:** https://console.mapbox.com

**Map Style URL:** `mapbox://styles/leotoke/cma4imtwv002u01sdg81ye34g`

| Credential | Environment |
| ---------- | ----------- |
| API Key    | Dev         |
| API Key    | Live        |
| Token      | Dev         |
| Token      | Live        |

***

## SendGrid

**Purpose:** Send verification and transactional emails.

**Console:** https://app.sendgrid.com

| Credential | Environment |
| ---------- | ----------- |
| API Key    | Single      |

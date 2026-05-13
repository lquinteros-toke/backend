---
icon: pagelines
---

# Strains

How the Flutter app fetches the strain variant customer table. The call goes from Flutter to Gateway, which proxies to BudHub and returns the data.

## Overview

The strain customer table contains pre-computed strain data optimized for the consumer app. It includes cannabinoids, terpenes, feelings, and basic strain info. The data lives in BudHub and is accessed through Gateway using the generic BudHub proxy endpoint.

Flutter sends a POST to Gateway with the target BudHub path and any filters. Gateway forwards the request to BudHub as a GET, then returns the response.

## Flow Overview

```
Flutter → Gateway POST /budhub (gateway_token)
           │
           ├─ Validates gateway_token
           ├─ Forwards to BudHub GET /strain_variant_customer_table
           ├─ Passes query_params as GET params
           │
           ▼
         Returns strain variant customer table data
```

## Endpoints

### POST /budhub - Multiple strain\_variants

Generic proxy endpoint that forwards requests to BudHub. Used for the strain customer table and other BudHub data calls.

**Called on:** Gateway directly from Flutter

**Auth:** `gateway_token`

**Headers:**

| Header          | Value                    | Required |
| --------------- | ------------------------ | -------- |
| `Authorization` | `Bearer <gateway_token>` | Yes      |
| `Content-Type`  | `application/json`       | Yes      |
| `X-Data-Source` | `test` or `live`         | Yes      |
| `X-Branch`      | `dev` or `v1`            | Yes      |

**Body:**

| Field          | Type | Required | Description                                                                                               |
| -------------- | ---- | -------- | --------------------------------------------------------------------------------------------------------- |
| `url`          | Text | Yes      | Full BudHub endpoint URL:  `https://xads-j1s5-11eg.f2.xano.io/api:f9SLs-_n/strain_variant_customer_table` |
| `query_params` | JSON | No       | Any filters or params to forward to BudHub                                                                |
| `branch`       | Text | No       | Branch override for the BudHub call                                                                       |

**query\_params for strain customer table:**

| Field           | Type    | Required | Description                                             |
| --------------- | ------- | -------- | ------------------------------------------------------- |
| `first_letter`  | Text    | No       | Filter strains by their starting letter (e.g. `A`, `B`) |
| `active`        | Text    | No       | Filter by active status (`true` or `false`)             |
| `language_code` | Text    | No       | Language code for translations. Defaults to `en`        |
| `page`          | Integer | No       | Page number (1-based). Defaults to 1                    |
| `per_page`      | Integer | No       | Number of results per page. Defaults to 20              |
| search\_query   | Text    | No       | Filter by word matching. Supports multiple languages    |

**Example request:**

```http
POST https://xads-j1s5-11eg.f2.xano.io/api:oGtZdRL8/budhub

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev

Body:
{
  "url": "https://xads-j1s5-11eg.f2.xano.io/api:f9SLs-_n/strain_variant_customer_table",
  "query_params": {
    "first_letter": "A",
    "active": "true",
    "language_code": "en",
    "page": 1,
    "per_page": 20,
    "search_query": "Pineapple"
  }
}
```

**Response:**

Returns an array of strain variant customer table records.

```json
{
"itemsReceived": 7,
"curPage": 1,
"nextPage": null,
"prevPage": null,
"offset": 0,
"itemsTotal": 7,
"pageTotal": 1,
  "items": [
    {
      "id": 1513,
      "created_at": 1767381492109,
      "strain_variant_id": 8408,
      "review_count": 0,
      "review_avg": 0,
      "product_count": 0,
      "first_letter": "A",
      "active": true,
      "popular": false,
      "featured": false,
      "image_url": "https://d20cab0473a264b1bd7335f5d5d3bcf1.cdn.bubble.io/f1766395830149x152897569082178100/ak-47.png",
      "strain_name": "AK-47",
      "strain_type_name": "Hybrid",
      "strain_type_empty_img_url": "https://d20cab...No%20Strain%20Image%20-%20Hybrid.png",
      "strain_cannabis_type_id": 3,
      "strain_cannabis_type_name": "Recreational",
      "strain_language_id": [
        {
          "id": 3904,
          "created_at": 1765486105937,
          "strain_id": 3843,
          "language_id": 1,
          "description": "AK-47 is a well-known hybrid strain famous for its balanced effects...",
          "name": "AK-47",
          "language_short_code": "en",
          "first_letter": "A"
        }
      ],
      "terpenes": [
        {
          "terpene_id": 6,
          "terpene_short_name": "β-Pinene",
          "mg_per_g": 27.57,
          "terpene_image": ""
        },
        {
          "terpene_id": 11,
          "terpene_short_name": "Caryophyllene",
          "mg_per_g": 2.06,
          "terpene_image": ""
        }
      ],
      "cannabinoids": [
        {
          "cannabinoid_id": 6,
          "cannabinoid_short_name": "CBD",
          "percentage": 94.15,
          "cannabinoid_image": ""
        },
        {
          "cannabinoid_id": 16,
          "cannabinoid_short_name": "CBG",
          "percentage": 11.7,
          "cannabinoid_image": ""
        },
        {
          "cannabinoid_id": 21,
          "cannabinoid_short_name": "THC",
          "percentage": 11.41,
          "cannabinoid_image": "https://xads-j1s5-11eg.f2.xano.io/vault/.../THC.svg"
        }
      ],
      "feelings": [
        {
          "feeling_id": 1,
          "feeling_name": "Energetic",
          "intensity": 3.44,
          "population": 28.6,
          "icon_url": "https://xads-j1s5-11eg.f2.xano.io/vault/.../Energetic.png"
        },
        {
          "feeling_id": 2,
          "feeling_name": "Aroused",
          "intensity": 2.55,
          "population": 31.9,
          "icon_url": "https://xads-j1s5-11eg.f2.xano.io/vault/.../Aroused.png"
        },
        {
          "feeling_id": 3,
          "feeling_name": "Euphoric",
          "intensity": 1.77,
          "population": 25,
          "icon_url": "https://xads-j1s5-11eg.f2.xano.io/vault/.../Euphoric.png"
        }
      ]
    }
  ]
}
```

| Field        | Type    | Description                                                     |
| ------------ | ------- | --------------------------------------------------------------- |
| `items`      | Array   | The strain records for the current page                         |
| `curPage`    | Integer | Current page number                                             |
| `itemsTotal` | Integer | Total number of strains matching the filters (across all pages) |



**Errors:**

| Status | Message                             | When                              |
| ------ | ----------------------------------- | --------------------------------- |
| `401`  | Unable to load and verify the token | Invalid or missing gateway\_token |
| `400`  | BudHub HTTP status code             | BudHub returned an error          |

***

## Language behavior

When `language_code` is `en` (or empty), results are returned directly with English strain names.

When a different `language_code` is provided (e.g. `es`, `fr`), the endpoint returns translated strain names where available. Strains without a translation in the requested language fall back to English.

## Filtering

All filters are optional. When omitted, all records are returned.

| Filter          | Effect                                                            |
| --------------- | ----------------------------------------------------------------- |
| `first_letter`  | Returns only strains starting with this letter                    |
| `active`        | `true` returns only active strains, `false` returns only inactive |
| `language_code` | Controls which language translation is applied to strain names    |
| `page`          | Which page of results to return (1-based, default `1`)            |
| `per_page`      | How many results per page (default `20`)                          |
| `search_query`  | Returns only strains matching this word in their name.            |

Filters can be combined. For example, `first_letter: "A"` + `active: "true"` + `page: 2` returns the second page of active strains starting with A.

{% hint style="info" %}
Pagination only applies to the customer table (list) endpoint. The single strain variant (detail) endpoint returns all matching records without pagination.
{% endhint %}

***

### POST /budhub — Single Strain Variant

Fetches detailed data for a single strain variant, including full compound details, datasource configurations, feelings, profiles, side effects, uses, activities, and translations

**Auth:** Gateway token

**Headers:**

| Header          | Value                    | Required |
| --------------- | ------------------------ | -------- |
| `Authorization` | `Bearer <gateway_token>` | Yes      |
| `Content-Type`  | `application/json`       | Yes      |
| `X-Data-Source` | `test` or `live`         | Yes      |
| `X-Branch`      | `dev` or `v1`            | Yes      |

**Body:**

| Field          | Type | Required | Description                                                                               |
| -------------- | ---- | -------- | ----------------------------------------------------------------------------------------- |
| `url`          | Text | Yes      | Full BudHub endpoint URL: `https://xads-j1s5-11eg.f2.xano.io/api:f9SLs-_n/strain_variant` |
| `query_params` | JSON | Yes      | Filters for the strain variant query                                                      |
| `branch`       | Text | No       | Branch override for the BudHub call                                                       |

**query\_params:**

| Field        | Type    | Required | Description                                                   |
| ------------ | ------- | -------- | ------------------------------------------------------------- |
| `variant_id` | Integer | No       | The strain variant ID to fetch                                |
| `strain_id`  | Integer | No       | The strain ID to fetch all its variants. Set null if not used |
| `active`     | Text    | No       | Filter by active status (`true` or `false`)                   |

> Use `variant_id` to fetch a specific variant. Use `strain_id` to fetch all variants for a strain. Both are optional but at least one should be provided.

**Example request:**

```
POST https://xads-j1s5-11eg.f2.xano.io/api:oGtZdRL8/budhub

Headers:
  Authorization: Bearer eyJhbGciOiJBMjU2S1ci...
  Content-Type: application/json
  X-Data-Source: test
  X-Branch: dev

Body:
{
  "url": "https://xads-j1s5-11eg.f2.xano.io/api:f9SLs-_n/strain_variant",
  "query_params": {
    "strain_id": null,
    "active": true,
    "variant_id": 8408
  }
}
```

**Response:**

Returns an array with the matching strain variant(s) including full compound data, datasource configs, temp arrays for feelings/profiles/side effects/uses/activities, and related lookups.

json

```json
[
  {
    "id": 12033,
    "created_at": 1766770395504,
    "strain_id": 3843,
    "strain_type": 1,
    "cannabis_type": 1,
    "active": false,
    "category_sub": 117,
    "popular": false,
    "featured": false,
    "condition": [],
    "profiles": [],
    "feelings": [],
    "side_effects": [],
    "uses": [],
    "strain_languages": [],
    "profiles_count": 1,
    "feelings_count": 1,
    "side_effects_count": 0,
    "uses_count": 1,
    "languages_count": 0,
    "terpenes_count": 1,
    "cannabinoids_count": 1,
    "strain_name_text": "AK-47",
    "first_letter": "A",
    "strain_variant_datasource_id": [],
    "cannabinoids_array": [],
    "terpenes_array": [],
    "requested_date": null,
    "information": 4,
    "category_id": 2,
    "strain_variant_admin_table_id": 3641,
    "strain_variant_customer_table_id": 1152,
    "cannabinoids": [
      {
        "cannabinoid_id": 1,
        "percentage": 23,
        "_cannabinoid": {
          "name": "Cannabigerovarinic Acid",
          "short_name": "CBGVA",
          "ignition_temp": 121,
          "melt_temp": 166,
          "burn_temp": 211,
          "cannabinoid_languages": [
            {
              "name": "",
              "short_name": "",
              "language_short_code": "en"
            },
            {
              "name": "Ácido cannabigerovarínico",
              "short_name": "CBGVA",
              "language_short_code": "es"
            }
          ],
          "profiles": [
            {
              "name": "Peppery",
              "font_color": "#5F5F5F",
              "background_color": "#E4E6E9",
              "icon": {
                "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/pQNAs3VHA2TlCGuxVycnI0NWVsg/c35cSQ../Property%25201%253DPeppery%25204.svg"
              }
            },
            {
              "name": "Coffee",
              "font_color": "#86572E",
              "background_color": "#EFDDC9",
              "icon": {
                "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/UPDT-wq1toSbLSKaRFT6HEQHvKQ/QFn1Bg../Property%252525201%2525253DCoffee.svg"
              }
            }
          ]
        }
      }
    ],
    "terpenes": [
      {
        "terpene_id": 3,
        "mg_per_g": 3,
        "_terpene": {
          "name": "β-Caryophyllene",
          "short_name": "β-Caryophyllene",
          "ignition_temp": 219,
          "melt_temp": 264,
          "burn_temp": 309,
          "terpene_languages": [
            {
              "terpene_id": 3,
              "name": "β-Caryophyllene",
              "short_name": "β-Caryophyllene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 3,
              "name": "Beta-cariofileno",
              "short_name": "β-Caryophyllene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      }
    ],
    "feelings_temp": [
      {
        "feeling_id": 2,
        "intensity": 3,
        "population": 23,
        "_feeling": {
          "name": "Aroused",
          "Description": "Test abc",
          "Icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/ybjKga0SD0ZQEq_NyoW5qCzq_zo/jpL9Bg../Aroused.png"
          }
        }
      }
    ],
    "profiles_temp": [
      {
        "profile_id": 20,
        "intensity": 3,
        "population": 0.3,
        "_profile": {
          "name": "Berry",
          "font_color": "#DA2841",
          "background_color": "#F7D8DD",
          "description": "The berry flavour in cannabis describes strains that emit aromas and tastes reminiscent of fresh, ripe berries, often a mix of sweet, tart, and sometimes earthy notes",
          "icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/-h3CJsTQICdZD4TCfTFY0inWPC4/IWliYQ../Property%2525201%25253DBerry.svg"
          }
        }
      }
    ],
    "side_effects_temp": [],
    "uses_temp": [],
    "_cannabis_type": {
      "name": "CBD",
      "short_name": "CBD",
      "font_color": "#3EA1F3",
      "background_color": "#F0F8FE"
    },
    "_category_sub": {
      "name": "Flowers"
    },
    "_strain": {
      "direct_akas": [
        "Hi"
      ],
      "genetic_akas": [
        "White Widow"
      ],
      "strain_parent_ids": [
      1,
      2
      ]
      "strain_languages_id": [
        {
          "language_id": 1,
          "description": "AK-47 is a well-known hybrid strain famous for its balanced effects. It delivers an uplifting and creative mental buzz while keeping the body relaxed, making it great for daytime or social use.",
          "name": "AK-47"
        },
        {
          "language_id": 13,
          "description": "AK-47 es una variedad híbrida muy conocida por sus efectos equilibrados. Ofrece una sensación mental estimulante y creativa, junto con una relajación corporal suave, ideal para usar durante el día o en situaciones sociales.",
          "name": "AK-47"
        }
      ],
      "strain_type_id": 1,
      "images_url": [
        "https://d20cab0473a264b1bd7335f5d5d3bcf1.cdn.bubble.io/f1766395830149x152897569082178100/ak-47.png"
      ]
    },
    "_strain_type": {
      "name": "Hybrid",
      "empty_image": "https://d20cab0473a264b1bd7335f5d5d3bcf1.cdn.bubble.io/f1767012332506x958173064905559000/No%20Strain%20Image%20-%20Hybrid.png"
    }
  },
  {
    "id": 8408,
    "created_at": 1765486105946,
    "strain_id": 3843,
    "strain_type": 1,
    "cannabis_type": 3,
    "active": true,
    "category_sub": 117,
    "popular": false,
    "featured": false,
    "condition": [],
    "profiles": [],
    "feelings": [],
    "side_effects": [],
    "uses": [],
    "strain_languages": [],
    "profiles_count": 0,
    "feelings_count": 0,
    "side_effects_count": 0,
    "uses_count": 0,
    "languages_count": 0,
    "terpenes_count": 7,
    "cannabinoids_count": 9,
    "strain_name_text": "AK-47",
    "first_letter": "A",
    "strain_variant_datasource_id": [
      {
        "id": 64,
        "created_at": 1765288894668,
        "strain_variants_id": 4761,
        "element": "uses",
        "source": "Weighted",
        "weight_static": 10,
        "weight_market": 90,
        "category_sub_id": 117,
        "cannabinoids": [],
        "terpenes": [],
        "profiles": [],
        "feelings": [],
        "uses": [],
        "side_effects": []
      },
      {
        "id": 63,
        "created_at": 1765287291197,
        "strain_variants_id": 4761,
        "element": "cannabinoids",
        "source": "Weighted",
        "weight_static": 11,
        "weight_market": 89,
        "category_sub_id": 117,
        "cannabinoids": [],
        "terpenes": [],
        "profiles": [],
        "feelings": [],
        "uses": [],
        "side_effects": []
      }
    ],
    "cannabinoids_array": [
      21,
      6,
      16,
      23,
      17
    ],
    "terpenes_array": [
      5,
      24,
      3,
      25,
      2
    ],
    "requested_date": null,
    "information": 4,
    "category_id": 2,
    "strain_variant_admin_table_id": 7,
    "strain_variant_customer_table_id": 1513,
    "cannabinoids": [
      {
        "cannabinoid_id": 21,
        "percentage": 15,
        "_cannabinoid": {
          "name": "Tetrahydrocannabinol",
          "short_name": "THC",
          "ignition_temp": 163,
          "melt_temp": 208,
          "burn_temp": 253,
          "cannabinoid_languages": [
            {
              "name": "",
              "short_name": "THC",
              "language_short_code": "en"
            },
            {
              "name": "Tetrahidrocannabinol",
              "short_name": "THC",
              "language_short_code": "es"
            }
          ],
          "profiles": [
            {
              "name": "Watermelon",
              "font_color": "#E21E1E",
              "background_color": "#FAE4E4",
              "icon": {
                "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/eleHhUmf0n1ihOitn5YzXVvPv5k/lVWd3w../Frame%2525201484578586.svg"
              }
            }
          ]
        }
      },
      {
        "cannabinoid_id": 6,
        "percentage": 0,
        "_cannabinoid": {
          "name": "Cannabidiol",
          "short_name": "CBD",
          "ignition_temp": 126,
          "melt_temp": 171,
          "burn_temp": 216,
          "cannabinoid_languages": [
            {
              "name": "",
              "short_name": "CBD",
              "language_short_code": "en"
            },
            {
              "name": "Cannabidiol",
              "short_name": "CBD",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "cannabinoid_id": 16,
        "percentage": 1,
        "_cannabinoid": {
          "name": "Cannabigerol",
          "short_name": "CBG",
          "ignition_temp": 286,
          "melt_temp": 331,
          "burn_temp": 376,
          "cannabinoid_languages": [
            {
              "name": "",
              "short_name": "CBG",
              "language_short_code": "en"
            },
            {
              "name": "Cannabigerol",
              "short_name": "CBG",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "cannabinoid_id": 23,
        "percentage": 0,
        "_cannabinoid": {
          "name": "Cannabichromene",
          "short_name": "CBC",
          "ignition_temp": 176,
          "melt_temp": 221,
          "burn_temp": 266,
          "cannabinoid_languages": [
            {
              "name": "",
              "short_name": "CBC",
              "language_short_code": "en"
            },
            {
              "name": "Cannabicromeno",
              "short_name": "CBC",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "cannabinoid_id": 17,
        "percentage": 0,
        "_cannabinoid": {
          "name": "Tetrahydrocannabivarin",
          "short_name": "THCV",
          "ignition_temp": 109,
          "melt_temp": 154,
          "burn_temp": 199,
          "cannabinoid_languages": [
            {
              "name": "",
              "short_name": "THCV",
              "language_short_code": "en"
            },
            {
              "name": "Tetrahidrocannabivarina",
              "short_name": "THCV",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      }
    ],
    "terpenes": [
      {
        "terpene_id": 6,
        "mg_per_g": 99,
        "_terpene": {
          "name": "β-Pinene",
          "short_name": "β-Pinene",
          "ignition_temp": 153,
          "melt_temp": 198,
          "burn_temp": 243,
          "terpene_languages": [
            {
              "terpene_id": 6,
              "name": "β-Pinene",
              "short_name": "β-Pinene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 6,
              "name": "Beta-pineno",
              "short_name": "β-Pinene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "terpene_id": 11,
        "mg_per_g": 3.191,
        "_terpene": {
          "name": "Caryophyllene",
          "short_name": "Caryophyllene",
          "ignition_temp": 129,
          "melt_temp": 174,
          "burn_temp": 219,
          "terpene_languages": [
            {
              "terpene_id": 11,
              "name": "Caryophyllene",
              "short_name": "Caryophyllene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 11,
              "name": "Cariofileno",
              "short_name": "Caryophyllene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "terpene_id": 22,
        "mg_per_g": 1.231,
        "_terpene": {
          "name": "Humulene",
          "short_name": "Humulene",
          "ignition_temp": 259,
          "melt_temp": 304,
          "burn_temp": 349,
          "terpene_languages": [
            {
              "terpene_id": 22,
              "name": "Humulene",
              "short_name": "Humulene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 22,
              "name": "Humuleno",
              "short_name": "Humulene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "terpene_id": 24,
        "mg_per_g": 1.836,
        "_terpene": {
          "name": "Limonene",
          "short_name": "Limonene",
          "ignition_temp": 143,
          "melt_temp": 188,
          "burn_temp": 233,
          "terpene_languages": [
            {
              "terpene_id": 24,
              "name": "Limonene",
              "short_name": "Limonene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 24,
              "name": "Limoneno",
              "short_name": "Limonene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "terpene_id": 26,
        "mg_per_g": 5.428,
        "_terpene": {
          "name": "Myrcene",
          "short_name": "Myrcene",
          "ignition_temp": 277,
          "melt_temp": 322,
          "burn_temp": 367,
          "terpene_languages": [
            {
              "terpene_id": 26,
              "name": "Myrcene",
              "short_name": "Myrcene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 26,
              "name": "Mirceno",
              "short_name": "Myrcene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "terpene_id": 30,
        "mg_per_g": 23,
        "_terpene": {
          "name": "Ocimene",
          "short_name": "Ocimene",
          "ignition_temp": 182,
          "melt_temp": 227,
          "burn_temp": 272,
          "terpene_languages": [
            {
              "terpene_id": 30,
              "name": "Ocimene",
              "short_name": "Ocimene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 30,
              "name": "Ocimeno",
              "short_name": "Ocimene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "terpene_id": 37,
        "mg_per_g": 1.869,
        "_terpene": {
          "name": "Pinene",
          "short_name": "Pinene",
          "ignition_temp": 238,
          "melt_temp": 283,
          "burn_temp": 328,
          "terpene_languages": [
            {
              "terpene_id": 37,
              "name": "Pinene",
              "short_name": "Pinene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 37,
              "name": "Pineno",
              "short_name": "Pinene",
              "language_short_code": "es"
            }
          ],
          "profiles": [
            {
              "name": "Mossy",
              "font_color": "#0A9536",
              "background_color": "#F3F3F3",
              "icon": {
                "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/8fcbsQutRNENu35icC2Gwa1UAgQ/MVM7hA../Icon.svg"
              }
            }
          ]
        }
      }
    ],
    "feelings_temp": [
      {
        "feeling_id": 7,
        "intensity": 3.44,
        "population": 28.6,
        "_feeling": {
          "name": "Focused",
          "Description": "",
          "Icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/sdvPPDb2yCufwlX71RhTlCPpiw4/bNnfuw../Focused%2520New.png"
          }
        }
      },
      {
        "feeling_id": 8,
        "intensity": 2.55,
        "population": 31.9,
        "_feeling": {
          "name": "Giggly",
          "Description": "",
          "Icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/cUin4fUUwnrcjv8R5LZToLPbgtE/fivhiw../Giggly.png"
          }
        }
      },
      {
        "feeling_id": 7,
        "intensity": 1.77,
        "population": 25,
        "_feeling": {
          "name": "Focused",
          "Description": "",
          "Icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/sdvPPDb2yCufwlX71RhTlCPpiw4/bNnfuw../Focused%2520New.png"
          }
        }
      }
    ],
    "profiles_temp": [
      {
        "profile_id": 42,
        "intensity": 3.44,
        "population": 28.6,
        "_profile": {
          "name": "Mossy",
          "font_color": "#0A9536",
          "background_color": "#F3F3F3",
          "description": "Mossy",
          "icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/8fcbsQutRNENu35icC2Gwa1UAgQ/MVM7hA../Icon.svg"
          }
        }
      },
      {
        "profile_id": 41,
        "intensity": 2.55,
        "population": 31.9,
        "_profile": {
          "name": "Watermelon",
          "font_color": "#E21E1E",
          "background_color": "#FAE4E4",
          "description": "Watermelon",
          "icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/eleHhUmf0n1ihOitn5YzXVvPv5k/lVWd3w../Frame%2525201484578586.svg"
          }
        }
      },
      {
        "profile_id": 40,
        "intensity": 1.77,
        "population": 25,
        "_profile": {
          "name": "Chemical",
          "font_color": "#5F5F5F",
          "background_color": "#E4E6E9",
          "description": "The chemical flavour in cannabis refers to strong, often pungent and sharp, tastes and aromas that can range from fuel-like or gassy to savoury or even slightly ammoniacal. These notes were traditionally attributed to terpenes but are now known to be primarily caused by minor, non-terpenoid compounds called flavourants, specifically volatile sulphur compounds (VSCs) and indole derivatives",
          "icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/QKvC13OAkA4ZCSu7Bk3QoA_AddQ/FvPcnQ../Property%2525201%25253DChemical.svg"
          }
        }
      }
    ],
    "side_effects_temp": [
      {
        "side_effect_id": 30,
        "intensity": 3.28,
        "population": 28,
        "_side_effects": {
          "name": "Dry Mouth",
          "description": "Dry Mouth"
        }
      },
      {
        "side_effect_id": 29,
        "intensity": 2.51,
        "population": 30.9,
        "_side_effects": {
          "name": "Dry Eyes",
          "description": "Dry Eyes"
        }
      },
      {
        "side_effect_id": 34,
        "intensity": 1.82,
        "population": 28.9,
        "_side_effects": {
          "name": "Groggy",
          "description": "Groggy"
        }
      }
    ],
    "uses_temp": [
      {
        "use_id": 27,
        "intensity": 1.9,
        "population": 16,
        "_use": {
          "name": "Insomnia",
          "description": "",
          "type": ""
        }
      },
      {
        "use_id": 17,
        "intensity": 1.72,
        "population": 13.8,
        "_use": {
          "name": "Pain",
          "description": "",
          "type": ""
        }
      },
      {
        "use_id": 4,
        "intensity": 1.46,
        "population": 21.9,
        "_use": {
          "name": "Inflammation",
          "description": "",
          "type": ""
        }
      },
      {
        "use_id": 9,
        "intensity": 1.24,
        "population": 14.7,
        "_use": {
          "name": "Fatigue",
          "description": "",
          "type": ""
        }
      }
    ],
    "_cannabis_type": {
      "name": "Recreational",
      "short_name": "REC",
      "font_color": "#736FD0",
      "background_color": "#F0EFFA"
    },
    "_category_sub": {
      "name": "Flowers"
    },
    "_strain": {
      "direct_akas": [
        "Hi"
      ],
      "genetic_akas": [
        "White Widow"
      ],
      "strain_languages_id": [
        {
          "language_id": 1,
          "description": "AK-47 is a well-known hybrid strain famous for its balanced effects. It delivers an uplifting and creative mental buzz while keeping the body relaxed, making it great for daytime or social use.",
          "name": "AK-47"
        },
        {
          "language_id": 13,
          "description": "AK-47 es una variedad híbrida muy conocida por sus efectos equilibrados. Ofrece una sensación mental estimulante y creativa, junto con una relajación corporal suave, ideal para usar durante el día o en situaciones sociales.",
          "name": "AK-47"
        }
      ],
      "strain_type_id": 1,
      "images_url": [
        "https://d20cab0473a264b1bd7335f5d5d3bcf1.cdn.bubble.io/f1766395830149x152897569082178100/ak-47.png"
      ]
    },
    "_strain_type": {
      "name": "Hybrid",
      "empty_image": "https://d20cab0473a264b1bd7335f5d5d3bcf1.cdn.bubble.io/f1767012332506x958173064905559000/No%20Strain%20Image%20-%20Hybrid.png"
    }
  },
  {
    "id": 12010,
    "created_at": 1766395960548,
    "strain_id": 3843,
    "strain_type": 1,
    "cannabis_type": 2,
    "active": false,
    "category_sub": 117,
    "popular": false,
    "featured": false,
    "condition": [],
    "profiles": [],
    "feelings": [],
    "side_effects": [],
    "uses": [],
    "strain_languages": [],
    "profiles_count": 0,
    "feelings_count": 1,
    "side_effects_count": 0,
    "uses_count": 0,
    "languages_count": 0,
    "terpenes_count": 2,
    "cannabinoids_count": 1,
    "strain_name_text": "AK-47",
    "first_letter": "A",
    "strain_variant_datasource_id": [],
    "cannabinoids_array": [
      21,
      6,
      16,
      23,
      17
    ],
    "terpenes_array": [
      5,
      24,
      3,
      25,
      2
    ],
    "requested_date": null,
    "information": 4,
    "category_id": 2,
    "strain_variant_admin_table_id": 117,
    "strain_variant_customer_table_id": 1729,
    "cannabinoids": [
      {
        "cannabinoid_id": 2,
        "percentage": 23,
        "_cannabinoid": {
          "name": "Cannabidivarinic Acid",
          "short_name": "CBDVA",
          "ignition_temp": 146,
          "melt_temp": 191,
          "burn_temp": 236,
          "cannabinoid_languages": [
            {
              "name": "",
              "short_name": "CBDVA",
              "language_short_code": "en"
            },
            {
              "name": "Ácido cannabidivarínico",
              "short_name": "CBDVA",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      }
    ],
    "terpenes": [
      {
        "terpene_id": 2,
        "mg_per_g": 23,
        "_terpene": {
          "name": "α-Pinene",
          "short_name": "α-Pinene",
          "ignition_temp": 187,
          "melt_temp": 232,
          "burn_temp": 277,
          "terpene_languages": [
            {
              "terpene_id": 2,
              "name": "α-Pinene",
              "short_name": "α-Pinene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 2,
              "name": "Alfa-pineno",
              "short_name": "α-Pinene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      },
      {
        "terpene_id": 3,
        "mg_per_g": 3,
        "_terpene": {
          "name": "β-Caryophyllene",
          "short_name": "β-Caryophyllene",
          "ignition_temp": 219,
          "melt_temp": 264,
          "burn_temp": 309,
          "terpene_languages": [
            {
              "terpene_id": 3,
              "name": "β-Caryophyllene",
              "short_name": "β-Caryophyllene",
              "language_short_code": "en"
            },
            {
              "terpene_id": 3,
              "name": "Beta-cariofileno",
              "short_name": "β-Caryophyllene",
              "language_short_code": "es"
            }
          ],
          "profiles": []
        }
      }
    ],
    "feelings_temp": [
      {
        "feeling_id": 1,
        "intensity": 2,
        "population": 23,
        "_feeling": {
          "name": "Energetic",
          "Description": "Energetic description",
          "Icon": {
            "url": "https://xads-j1s5-11eg.f2.xano.io/vault/qZFvyyaC/xe1tzRvgAFeUKPUo7lMfPcQhwWI/oTejRA../Energetic.png"
          }
        }
      }
    ],
    "profiles_temp": [],
    "side_effects_temp": [],
    "uses_temp": [],
    "_cannabis_type": {
      "name": "Medical",
      "short_name": "MMJ",
      "font_color": "#4893A0",
      "background_color": "#E2F3F6"
    },
    "_category_sub": {
      "name": "Flowers"
    },
    "_strain": {
      "direct_akas": [
        "Hi"
      ],
      "genetic_akas": [
        "White Widow"
      ],
      "strain_languages_id": [
        {
          "language_id": 1,
          "description": "AK-47 is a well-known hybrid strain famous for its balanced effects. It delivers an uplifting and creative mental buzz while keeping the body relaxed, making it great for daytime or social use.",
          "name": "AK-47"
        },
        {
          "language_id": 13,
          "description": "AK-47 es una variedad híbrida muy conocida por sus efectos equilibrados. Ofrece una sensación mental estimulante y creativa, junto con una relajación corporal suave, ideal para usar durante el día o en situaciones sociales.",
          "name": "AK-47"
        }
      ],
      "strain_type_id": 1,
      "images_url": [
        "https://d20cab0473a264b1bd7335f5d5d3bcf1.cdn.bubble.io/f1766395830149x152897569082178100/ak-47.png"
      ]
    },
    "_strain_type": {
      "name": "Hybrid",
      "empty_image": "https://d20cab0473a264b1bd7335f5d5d3bcf1.cdn.bubble.io/f1767012332506x958173064905559000/No%20Strain%20Image%20-%20Hybrid.png"
    }
  }
]
```

**Errors:**

| Status | Message                             | When                              |
| ------ | ----------------------------------- | --------------------------------- |
| `401`  | Unable to load and verify the token | Invalid or missing gateway\_token |
| `400`  | BudHub HTTP status code             | BudHub returned an error          |

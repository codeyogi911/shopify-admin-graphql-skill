# Logistics — Delivery Profiles, Shipping Rates, Carrier Services, Markets

Patterns for shipping configuration (who ships what, from where, to where, at what price) and markets (which buyers see which catalog, currency, and domain).

---

## Delivery Profiles

A `DeliveryProfile` is the top-level container for "who ships what, where, at what rate." Every shop has a default profile; create additional profiles when a subset of products needs different shipping behavior (e.g., heavy freight, cold-chain, digital goods).

**Structure**:
- A profile contains one or more **location groups** — sets of warehouse/store locations that share a shipping configuration.
- Each location group contains one or more **zones** — geographic targets (e.g., "USA", "EU").
- Each zone contains one or more **method definitions** — the actual rates the buyer sees at checkout, each with a `rateDefinition` (flat price) or a `participant` (carrier-calculated rate).

**Required scope**: `write_shipping` (or `manage_delivery_settings` user permission).

### List delivery profiles

```graphql
query ListDeliveryProfiles($first: Int!, $merchantOwnedOnly: Boolean) {
  deliveryProfiles(first: $first, merchantOwnedOnly: $merchantOwnedOnly) {
    edges {
      node {
        id
        name
        default
        profileLocationGroups {
          locationGroup { id locations(first: 5) { edges { node { id name } } } }
          locationGroupZones(first: 10) {
            edges {
              node {
                zone { id name countries { code { countryCode } provinces { code } } }
                methodDefinitions(first: 10) {
                  edges {
                    node {
                      id
                      name
                      active
                      rateProvider {
                        ... on DeliveryRateDefinition { price { amount currencyCode } }
                        ... on DeliveryParticipant { carrierService { id name } }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

Pass `merchantOwnedOnly: true` to exclude profiles managed by third-party shipping apps (you rarely want to edit those).

### Create a delivery profile

```graphql
mutation CreateDeliveryProfile($profile: DeliveryProfileInput!) {
  deliveryProfileCreate(profile: $profile) {
    profile {
      id
      name
      profileLocationGroups {
        locationGroup { id }
        locationGroupZones(first: 10) { edges { node { zone { name } } } }
      }
    }
    userErrors { field message }
  }
}
```

**Variables example** (new "Heavy Freight" profile, ships to US from one warehouse at a flat $49):

```json
{
  "profile": {
    "name": "Heavy Freight",
    "profileLocationGroups": [
      {
        "locations": ["gid://shopify/Location/456"],
        "zonesToCreate": [
          {
            "name": "USA",
            "countries": [{ "code": "US", "includeAllProvinces": true }],
            "methodDefinitionsToCreate": [
              {
                "name": "Standard Freight",
                "active": true,
                "rateDefinition": {
                  "price": { "amount": "49.00", "currencyCode": "USD" }
                }
              }
            ]
          }
        ]
      }
    ],
    "variantsToAssociate": ["gid://shopify/ProductVariant/111", "gid://shopify/ProductVariant/222"]
  }
}
```

**Shopify's own advice**: start with ≤5 location groups in the create call. Add more later via `deliveryProfileUpdate` — deeply nested creates are fragile.

### Add a zone to an existing profile

```graphql
mutation AddZoneToProfile($id: ID!, $profile: DeliveryProfileInput!) {
  deliveryProfileUpdate(id: $id, profile: $profile) {
    profile { id name }
    userErrors { field message }
  }
}
```

**Variables example** (add a Canada zone with two tiers — free over $100, flat $12 otherwise — to an existing location group):

```json
{
  "id": "gid://shopify/DeliveryProfile/789",
  "profile": {
    "locationGroupsToUpdate": [
      {
        "id": "gid://shopify/DeliveryLocationGroup/999",
        "zonesToCreate": [
          {
            "name": "Canada",
            "countries": [{ "code": "CA", "includeAllProvinces": true }],
            "methodDefinitionsToCreate": [
              {
                "name": "Free over $100",
                "active": true,
                "rateDefinition": { "price": { "amount": "0.00", "currencyCode": "CAD" } },
                "conditionsToCreate": [
                  {
                    "field": "TOTAL_PRICE",
                    "operator": "GREATER_THAN_OR_EQUAL_TO",
                    "criteria": { "amount": "100.00", "currencyCode": "CAD" }
                  }
                ]
              },
              {
                "name": "Standard",
                "active": true,
                "rateDefinition": { "price": { "amount": "12.00", "currencyCode": "CAD" } }
              }
            ]
          }
        ]
      }
    ]
  }
}
```

**Condition fields**: `TOTAL_PRICE` or `TOTAL_WEIGHT`. **Operators**: `LESS_THAN_OR_EQUAL_TO`, `GREATER_THAN_OR_EQUAL_TO`.

---

## Carrier Services (Real-Time Rates)

A `CarrierService` is a webhook-style integration: when a buyer reaches shipping at checkout, Shopify POSTs the cart contents to your `callbackUrl`, waits for a JSON response with rate options, and renders them. Use this for real-time quotes from FedEx/UPS/DHL, 3PL-specific rates, or dimensional-weight calculators.

**Required scope**: `write_shipping`.

### Create a carrier service

```graphql
mutation CreateCarrierService($input: DeliveryCarrierServiceCreateInput!) {
  carrierServiceCreate(input: $input) {
    carrierService {
      id
      name
      callbackUrl
      active
      supportsServiceDiscovery
    }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "input": {
    "name": "Acme 3PL Real-Time",
    "callbackUrl": "https://shipping.acme.example/shopify/rates",
    "supportsServiceDiscovery": true,
    "active": true
  }
}
```

### What Shopify sends to your callback

Shopify POSTs JSON like this to `callbackUrl` at checkout time:

```json
{
  "rate": {
    "origin": { "country": "US", "postal_code": "78701", "province": "TX" },
    "destination": { "country": "US", "postal_code": "10001", "province": "NY" },
    "items": [
      { "name": "Merino Crew", "sku": "MC-CHA-M", "quantity": 2, "grams": 400, "price": 8900, "vendor": "Acme", "requires_shipping": true, "taxable": true }
    ],
    "currency": "USD",
    "locale": "en"
  }
}
```

Your service **must** respond within ~10 seconds with:

```json
{
  "rates": [
    {
      "service_name": "Ground",
      "service_code": "GROUND",
      "total_price": 1299,
      "currency": "USD",
      "min_delivery_date": "2026-04-15T00:00:00-07:00",
      "max_delivery_date": "2026-04-17T00:00:00-07:00"
    }
  ]
}
```

`total_price` is in the **smallest currency unit** (cents for USD). A timeout or non-200 response causes Shopify to hide the carrier service's rates for that checkout — no rates offered, not an error to the buyer.

### Attach a carrier service as a rate in a delivery profile

Reference the carrier service from a method definition via `participant` instead of `rateDefinition`:

```json
{
  "methodDefinitionsToCreate": [
    {
      "name": "Acme 3PL",
      "active": true,
      "participant": {
        "carrierServiceId": "gid://shopify/DeliveryCarrierService/42",
        "fixedFee": { "amount": "2.50", "currencyCode": "USD" },
        "percentageOfRateFee": 0,
        "participantServices": [{ "name": "GROUND", "active": true }]
      }
    }
  ]
}
```

`fixedFee` and `percentageOfRateFee` let you mark up the carrier's quoted rate — set both to zero to pass through exactly what the carrier returns.

---

## Markets

A `Market` is a container for **contextual buyer experience**: which countries see which currency, language, domain, catalog, and payment methods. Markets drive international expansion and B2B segmentation without creating separate stores.

**Required scopes**: `read_markets` + `write_markets`.

### List markets

```graphql
query ListMarkets($first: Int!, $query: String) {
  markets(first: $first, query: $query) {
    edges {
      node {
        id
        name
        handle
        enabled
        primary
        currencySettings {
          baseCurrency { currencyCode }
          localCurrencies
        }
        webPresence {
          id
          rootUrls { locale url }
          defaultLocale { locale }
          alternateLocales { locale }
        }
      }
    }
  }
}
```

**Filter examples**: `name:europe`, `type:regional`

**Sort keys**: `NAME`, `ID`

### Create a market

```graphql
mutation CreateMarket($input: MarketCreateInput!) {
  marketCreate(input: $input) {
    market {
      id
      name
      handle
      enabled
    }
    userErrors { field message code }
  }
}
```

**Variables example** (new EU market for France, Germany, and Spain):

```json
{
  "input": {
    "name": "Europe",
    "handle": "eu",
    "conditions": {
      "conditionsToAdd": {
        "regionsCondition": {
          "regions": [
            { "countryCode": "FR" },
            { "countryCode": "DE" },
            { "countryCode": "ES" }
          ]
        }
      }
    }
  }
}
```

> ⚠️ The older `regions` field on `MarketCreateInput` is **deprecated**. Add countries via `conditions.conditionsToAdd.regionsCondition.regions` instead, both on create and on update.

### Add or remove regions from an existing market

```graphql
mutation UpdateMarketRegions($id: ID!, $input: MarketUpdateInput!) {
  marketUpdate(id: $id, input: $input) {
    market { id name }
    userErrors { field message code }
  }
}
```

**Variables example** (add United Kingdom, remove Spain):

```json
{
  "id": "gid://shopify/Market/123",
  "input": {
    "conditions": {
      "conditionsToAdd": {
        "regionsCondition": { "regions": [{ "countryCode": "GB" }] }
      },
      "conditionsToDelete": {
        "regionsCondition": { "regions": [{ "countryCode": "ES" }] }
      }
    }
  }
}
```

> ⚠️ `marketRegionsCreate` (and its siblings `marketRegionsDelete`, `marketRegionDelete`) are **deprecated**. Use `marketUpdate` with `conditions.conditionsToAdd` / `conditionsToDelete` as shown above.

### Web presence (domain + locales)

A market's web presence is how it appears to shoppers: either a dedicated domain (`shop.fr`) or a subfolder on the primary domain (`shop.com/fr`), with one default locale and zero or more alternates.

```graphql
mutation CreateWebPresence($input: WebPresenceCreateInput!) {
  webPresenceCreate(input: $input) {
    webPresence {
      id
      rootUrls { locale url }
      defaultLocale { locale }
      alternateLocales { locale }
    }
    userErrors { field message code }
  }
}
```

> ⚠️ `marketWebPresenceCreate` is **deprecated**. Use `webPresenceCreate` (passes market IDs inside the input) going forward. The input shape lives under `WebPresenceCreateInput` — introspect it before shipping, since it evolved with the migration off the per-market mutation.

### Currency settings

Currency behavior lives under `MarketCurrencySettings`:

- `baseCurrency` — the single currency in which prices are stored for this market.
- `localCurrencies` (boolean) — when `true`, Shopify auto-converts displayed prices into the buyer's regional currency at checkout using Shopify's FX rates.

Update via `marketUpdate` with `currencySettings: { baseCurrency: "EUR", localCurrencies: true }`.

---

## How the pieces fit together

A buyer in Germany visiting `shop.com/de`:

1. **Market** matches on `DE` in its regions condition → binds the request to the "Europe" market.
2. The market's **web presence** resolves the subfolder/domain and picks the default locale (`de`).
3. The market's **catalog** (from [b2b.md](b2b.md) / `catalogCreate`) determines which products show and at what prices.
4. The market's **currency settings** set the base currency (EUR) and whether to display in the buyer's local currency.
5. At checkout, the **delivery profile** matching the cart's variants picks a zone containing `DE`, and its **method definitions** render — either flat rates or carrier-service rates.

# B2B — Companies, Locations, Catalogs, Price Lists, Payment Terms

Patterns for business-to-business commerce: managing company buyers, contextual pricing, and deferred payment.

> **Plan gate**: The B2B object model (`Company`, `CompanyLocation`, `CompanyContact`, `Catalog`) is **Shopify Plus only**. These mutations return errors on non-Plus stores even when scopes are granted.

**Required scopes** (varies by mutation):
- `read_companies` / `write_companies` — or the broader `read_customers` / `write_customers`
- `write_products` — for `priceListCreate` and price adjustments
- `write_publications` — for linking publications to catalogs

---

## Companies

A `Company` is the top-level B2B buyer. Each company has one or more **contacts** (people who place orders) and one or more **locations** (billing/shipping endpoints with their own catalogs and payment terms).

### List companies

```graphql
query ListCompanies($first: Int!, $after: String, $query: String) {
  companies(first: $first, after: $after, query: $query) {
    edges {
      node {
        id
        name
        externalId
        createdAt
        customerSince
        lifetimeDuration
        locationsCount { count }
        contactsCount { count }
        ordersCount { count }
        totalSpent { amount currencyCode }
        mainContact { customer { id displayName email } }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `name:Acme`, `external_id:ERP-12345`, `created_at:>2026-01-01`, `active_customers_count:>0`, `metafields.custom.tier:gold`

**Sort keys**: `ID`, `NAME`, `CREATED_AT`, `UPDATED_AT`, `RELEVANCE`

### Create company (+ initial contact + location)

`companyCreate` is a one-shot: it creates the company, the first contact, and the first location in a single call.

```graphql
mutation CreateCompany($input: CompanyCreateInput!) {
  companyCreate(input: $input) {
    company {
      id
      name
      mainContact { id customer { id email } }
      locations(first: 5) { edges { node { id name } } }
    }
    userErrors { field message code }
  }
}
```

**Variables example**:

```json
{
  "input": {
    "company": {
      "name": "Acme Wholesale",
      "externalId": "ERP-12345",
      "note": "Net-30 wholesale account"
    },
    "companyContact": {
      "email": "buyer@acme.example",
      "firstName": "Jamie",
      "lastName": "Chen",
      "title": "Purchasing Manager"
    },
    "companyLocation": {
      "name": "HQ — Austin",
      "billingAddress": {
        "address1": "500 Congress Ave",
        "city": "Austin",
        "zoneCode": "TX",
        "countryCode": "US",
        "zip": "78701"
      },
      "shippingAddress": {
        "address1": "500 Congress Ave",
        "city": "Austin",
        "zoneCode": "TX",
        "countryCode": "US",
        "zip": "78701"
      },
      "billingSameAsShipping": true
    }
  }
}
```

Error payloads use `BusinessCustomerUserError` (includes a `code` field in addition to `field` / `message`).

### Read a company with everything attached

```graphql
query GetCompany($id: ID!) {
  company(id: $id) {
    id
    name
    note
    totalSpent { amount currencyCode }
    contacts(first: 20) {
      edges { node { id customer { id displayName email } title } }
    }
    locations(first: 20) {
      edges {
        node {
          id
          name
          billingAddress { formatted }
          shippingAddress { formatted }
          buyerExperienceConfiguration { paymentTermsTemplate { id name } checkoutToDraft editableShippingAddress }
        }
      }
    }
    orders(first: 10, reverse: true) {
      edges { node { id name totalPriceSet { shopMoney { amount currencyCode } } } }
    }
    draftOrders(first: 10) {
      edges { node { id name status } }
    }
  }
}
```

---

## Company Locations

A `CompanyLocation` is where pricing, payment terms, and tax behavior actually attach. A company with three branches buying at different prices has three locations.

### Create a location under an existing company

```graphql
mutation CreateCompanyLocation($companyId: ID!, $input: CompanyLocationInput!) {
  companyLocationCreate(companyId: $companyId, input: $input) {
    companyLocation {
      id
      name
      buyerExperienceConfiguration { checkoutToDraft editableShippingAddress }
    }
    userErrors { field message code }
  }
}
```

**Variables example**:

```json
{
  "companyId": "gid://shopify/Company/123",
  "input": {
    "name": "Warehouse — Reno",
    "billingAddress": {
      "address1": "900 Industrial Way",
      "city": "Reno",
      "zoneCode": "NV",
      "countryCode": "US",
      "zip": "89502"
    },
    "shippingAddress": {
      "address1": "900 Industrial Way",
      "city": "Reno",
      "zoneCode": "NV",
      "countryCode": "US",
      "zip": "89502"
    },
    "billingSameAsShipping": true,
    "locale": "en",
    "taxRegistrationId": "NV-12345678"
  }
}
```

### buyerExperienceConfiguration — controls checkout behavior

- `checkoutToDraft: true` — orders placed by this location become draft orders first (merchant approves/edits before charging).
- `editableShippingAddress: false` — locks the buyer to the addresses on file.
- `paymentTermsTemplateId` — default payment terms for every order placed by this location (see Payment Terms below).

---

## Catalogs

A `Catalog` binds **products + pricing** to a **context** (one or more company locations, or a market). It's the glue between B2B locations and contextual pricing.

Two usage modes:
1. **Pricing-only**: location sees the full storefront product list but at custom prices (via `priceListId`).
2. **Pricing + visibility**: location sees a curated subset of products (via `publicationId`) at custom prices.

### Create a catalog

```graphql
mutation CreateCatalog($input: CatalogCreateInput!) {
  catalogCreate(input: $input) {
    catalog {
      id
      title
      status
      priceList { id name currency }
      publication { id name }
      ... on CompanyLocationCatalog {
        companyLocations(first: 10) { edges { node { id name } } }
      }
    }
    userErrors { field message code }
  }
}
```

**Variables example** (pricing-only catalog for a single location):

```json
{
  "input": {
    "title": "Acme Reno — Wholesale Pricing",
    "status": "ACTIVE",
    "context": {
      "companyLocationIds": ["gid://shopify/CompanyLocation/456"]
    },
    "priceListId": "gid://shopify/PriceList/789"
  }
}
```

Omit `publicationId` for pricing-only catalogs — product visibility then follows the default sales channel rules.

`CatalogType` variants returned by the API include `CompanyLocationCatalog` (B2B) and `MarketCatalog` (markets/localization). Use inline fragments when reading the `catalog` field.

### List catalogs for a company location

```graphql
query CompanyLocationCatalogs($id: ID!) {
  companyLocation(id: $id) {
    id
    name
    catalogs(first: 10) {
      edges {
        node {
          id
          title
          status
          priceList { id name currency }
        }
      }
    }
  }
}
```

---

## Price Lists

A `PriceList` stores custom prices in a single currency. It's either a **percentage adjustment** off the base catalog price, or a table of **fixed per-variant prices** (or both). Price lists are attached to locations via catalogs.

### Create a price list with a percentage adjustment

```graphql
mutation CreatePriceList($input: PriceListCreateInput!) {
  priceListCreate(input: $input) {
    priceList {
      id
      name
      currency
      parent { adjustment { type value } }
    }
    userErrors { field message code }
  }
}
```

**Variables example** (15% off the base catalog):

```json
{
  "input": {
    "name": "Wholesale 15% Off",
    "currency": "USD",
    "parent": {
      "adjustment": {
        "type": "PERCENTAGE_DECREASE",
        "value": 15.0
      }
    }
  }
}
```

`PriceListAdjustmentType` enum: `PERCENTAGE_DECREASE`, `PERCENTAGE_INCREASE`.

### Add fixed per-variant prices

Use this when specific SKUs need a hand-set wholesale price that isn't a simple percentage off retail.

```graphql
mutation AddFixedPrices($priceListId: ID!, $prices: [PriceListPriceInput!]!) {
  priceListFixedPricesAdd(priceListId: $priceListId, prices: $prices) {
    prices {
      variant { id sku }
      price { amount currencyCode }
      compareAtPrice { amount currencyCode }
    }
    userErrors { field message code }
  }
}
```

**Variables example**:

```json
{
  "priceListId": "gid://shopify/PriceList/789",
  "prices": [
    {
      "variantId": "gid://shopify/ProductVariant/111",
      "price": { "amount": "45.00", "currencyCode": "USD" },
      "compareAtPrice": { "amount": "89.00", "currencyCode": "USD" }
    },
    {
      "variantId": "gid://shopify/ProductVariant/222",
      "price": { "amount": "22.00", "currencyCode": "USD" }
    }
  ]
}
```

The mutation is upsert-style: existing fixed prices for those variants are replaced. Currency on every entry must match the price list's `currency`, or you'll get a `userErrors` entry.

To remove fixed prices, use `priceListFixedPricesDelete(priceListId:, variantIds:)`.

---

## Payment Terms

Payment terms defer payment on an order (Net-30, due on receipt, pay on a fixed date). In B2B flows they're usually pre-configured on a company location's `buyerExperienceConfiguration.paymentTermsTemplate`, but you can also attach terms to an individual order or draft order via `paymentTermsCreate`.

**Required scopes**: `write_payment_terms` (or `write_orders`).

### Attach payment terms to an order

```graphql
mutation CreatePaymentTerms($referenceId: ID!, $paymentTermsAttributes: PaymentTermsCreateInput!) {
  paymentTermsCreate(referenceId: $referenceId, paymentTermsAttributes: $paymentTermsAttributes) {
    paymentTerms {
      id
      paymentTermsName
      paymentTermsType
      dueInDays
      overdue
      paymentSchedules(first: 5) {
        edges { node { id dueAt completedAt amount { amount currencyCode } } }
      }
    }
    userErrors { field message code }
  }
}
```

**Variables example** (Net-30 on an existing order):

```json
{
  "referenceId": "gid://shopify/Order/123",
  "paymentTermsAttributes": {
    "paymentTermsTemplateId": "gid://shopify/PaymentTermsTemplate/2",
    "paymentSchedules": [
      { "issuedAt": "2026-04-11T00:00:00Z" }
    ]
  }
}
```

**`PaymentTermsType` values** (from the built-in templates):
- `RECEIPT` — due immediately on receipt of the invoice.
- `NET` — due N days after issue (common: Net-7, Net-15, Net-30, Net-45, Net-60, Net-90).
- `FIXED` — due on a specific calendar date.
- `UNKNOWN` — fallback for imported orders.

List the available templates via `paymentTermsTemplates` to get the right `paymentTermsTemplateId` for each type.

### Query templates

```graphql
query ListPaymentTermsTemplates {
  paymentTermsTemplates {
    id
    name
    description
    paymentTermsType
    dueInDays
  }
}
```

---

## B2B reorder flow — `draftOrderCreateFromOrder`

Common B2B pattern: buyer asks the merchant to "reorder what we bought last quarter." Convert a historical order directly into a new draft order that the buyer can review and adjust.

**Required scopes**: `write_draft_orders` + `write_orders`.

```graphql
mutation ReorderAsDraft($orderId: ID!) {
  draftOrderCreateFromOrder(orderId: $orderId) {
    draftOrder {
      id
      name
      invoiceUrl
      totalPriceSet { shopMoney { amount currencyCode } }
    }
    userErrors { field message }
  }
}
```

The resulting draft order inherits line items, customer, and addresses from the source order. Edit via the standard draft order mutations, then `draftOrderComplete` to convert to a real order.

---

## Putting it together — a typical B2B onboarding flow

1. **`companyCreate`** with the buyer's info + first contact + first location.
2. **`priceListCreate`** with a percentage adjustment (or skip, and use fixed prices only).
3. **`priceListFixedPricesAdd`** for any SKUs that need hand-set wholesale prices.
4. **`catalogCreate`** binding the price list (and optionally a publication for curated visibility) to the company location via `context.companyLocationIds`.
5. Configure `buyerExperienceConfiguration.paymentTermsTemplateId` on the location so orders inherit Net-30 (or similar) by default.
6. When the buyer places an order, it automatically uses the catalog's prices and the location's payment terms.

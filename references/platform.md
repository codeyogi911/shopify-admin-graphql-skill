# Platform — Bulk Operations, Webhooks, Access Scopes

Patterns for platform-level concerns: bulk data export, webhook subscriptions, and scope reference.

---

## Bulk Operations (for large datasets)

```graphql
mutation BulkQuery {
  bulkOperationRunQuery(
    query: """
    {
      products {
        edges {
          node {
            id
            title
            variants {
              edges {
                node { id sku price inventoryQuantity }
              }
            }
          }
        }
      }
    }
    """
  ) {
    bulkOperation { id status }
    userErrors { field message }
  }
}
```

Poll for completion:

```graphql
query BulkOperationStatus {
  currentBulkOperation {
    id
    status
    errorCode
    objectCount
    url
  }
}
```

The `url` field returns a JSONL file download link when status is `COMPLETED`.

---

## Webhooks

### Register a webhook

```graphql
mutation CreateWebhook($topic: WebhookSubscriptionTopic!, $webhookSubscription: WebhookSubscriptionInput!) {
  webhookSubscriptionCreate(topic: $topic, webhookSubscription: $webhookSubscription) {
    webhookSubscription { id topic endpoint { ... on WebhookHttpEndpoint { callbackUrl } } }
    userErrors { field message }
  }
}
```

Common topics: `ORDERS_CREATE`, `ORDERS_UPDATED`, `PRODUCTS_CREATE`, `PRODUCTS_UPDATE`, `PRODUCTS_DELETE`, `CUSTOMERS_CREATE`, `CUSTOMERS_UPDATE`, `INVENTORY_LEVELS_UPDATE`, `APP_UNINSTALLED`

---

## Access Scopes Reference

| Scope | Access |
|---|---|
| `read_products` / `write_products` | Products, variants, collections |
| `read_orders` / `write_orders` | Orders, transactions, fulfillments |
| `read_customers` / `write_customers` | Customers, addresses |
| `read_inventory` / `write_inventory` | Inventory items, levels |
| `read_discounts` / `write_discounts` | Discount codes, automatic discounts |
| `read_fulfillments` / `write_fulfillments` | Fulfillments, tracking |
| `read_shipping` / `write_shipping` | Shipping zones, rates |
| `read_content` / `write_content` | Pages, blogs, articles |
| `read_themes` / `write_themes` | Theme files |
| `read_locales` / `write_locales` | Translations |

---

## App Billing (for app developers)

Shopify handles app monetization through **app subscriptions** — recurring, usage-based, or combined plans that charge the merchant on behalf of your app. The flow is:

1. Call `appSubscriptionCreate` → receive a `confirmationUrl`.
2. Redirect the merchant to `confirmationUrl`. They approve or decline the charge in the Shopify admin.
3. Shopify sends the merchant back to your `returnUrl` and sets the subscription to `ACTIVE` on approval.
4. For usage-based plans, your app calls `appUsageRecordCreate` whenever a billable event occurs.

**Required scope**: no explicit scope; billing works via the app's own API credentials. Requires a published, installed app.

### Inspect the current install & active plan

```graphql
query GetCurrentInstallation {
  currentAppInstallation {
    id
    launchUrl
    uninstallUrl
    accessScopes { handle }
    activeSubscriptions {
      id
      name
      status
      currentPeriodEnd
      trialDays
      test
      lineItems {
        id
        plan {
          pricingDetails {
            __typename
            ... on AppRecurringPricing {
              interval
              price { amount currencyCode }
              discount { durationLimitInIntervals remainingDurationInIntervals value { ... on AppSubscriptionDiscountAmount { amount { amount currencyCode } } ... on AppSubscriptionDiscountPercentage { percentage } } }
            }
            ... on AppUsagePricing {
              terms
              cappedAmount { amount currencyCode }
              balanceUsed { amount currencyCode }
            }
          }
        }
      }
    }
  }
}
```

Call this on every merchant load to verify the install is still active and the plan matches what your app expects.

### Create a subscription (recurring + usage)

`appSubscriptionCreate` takes **top-level scalar arguments**, not a wrapper input object.

```graphql
mutation CreateAppSubscription(
  $name: String!
  $returnUrl: URL!
  $lineItems: [AppSubscriptionLineItemInput!]!
  $trialDays: Int
  $test: Boolean
) {
  appSubscriptionCreate(
    name: $name
    returnUrl: $returnUrl
    lineItems: $lineItems
    trialDays: $trialDays
    test: $test
  ) {
    appSubscription { id name status }
    confirmationUrl
    userErrors { field message }
  }
}
```

**Variables example** (Pro plan: $29.99/month flat + $0.05 per API call, capped at $200/month, 14-day trial):

```json
{
  "name": "Pro Plan",
  "returnUrl": "https://app.example.com/shopify/billing/callback",
  "trialDays": 14,
  "test": false,
  "lineItems": [
    {
      "plan": {
        "appRecurringPricingDetails": {
          "interval": "EVERY_30_DAYS",
          "price": { "amount": "29.99", "currencyCode": "USD" }
        }
      }
    },
    {
      "plan": {
        "appUsagePricingDetails": {
          "terms": "$0.05 per API call",
          "cappedAmount": { "amount": "200.00", "currencyCode": "USD" }
        }
      }
    }
  ]
}
```

**Key points**:
- **Only one of each plan type** per subscription — one recurring plan and one usage plan max.
- `interval` enum: `EVERY_30_DAYS`, `ANNUAL`.
- **Set `test: true` during development** — charges on test subscriptions never hit the merchant's card. Production subscriptions with `test: true` are rejected by Shopify's review process, so flip it back before shipping.
- **`confirmationUrl`** is single-use. If the merchant closes the tab without approving, generate a new subscription to retry.

### Record a usage charge

```graphql
mutation RecordUsage(
  $subscriptionLineItemId: ID!
  $description: String!
  $price: MoneyInput!
  $idempotencyKey: String
) {
  appUsageRecordCreate(
    subscriptionLineItemId: $subscriptionLineItemId
    description: $description
    price: $price
    idempotencyKey: $idempotencyKey
  ) {
    appUsageRecord {
      id
      description
      price { amount currencyCode }
    }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "subscriptionLineItemId": "gid://shopify/AppSubscriptionLineItem/456?v=1",
  "description": "2,000 API calls — 2026-04-11",
  "price": { "amount": "100.00", "currencyCode": "USD" },
  "idempotencyKey": "usage-2026-04-11-acme-shop"
}
```

**`idempotencyKey`** caps at 255 chars. Always set it when the charge has a natural unique identifier (a billing cycle, an invoice number) so retries don't double-charge.

**Cap enforcement**: if the new charge would push the billing interval's total usage over `cappedAmount`, the mutation errors. Either raise the cap via `appSubscriptionLineItemUpdate` (triggers a re-confirmation flow with the merchant) or reject the billable action in your app.

### Cancel a subscription

```graphql
mutation CancelAppSubscription($id: ID!, $prorate: Boolean) {
  appSubscriptionCancel(id: $id, prorate: $prorate) {
    appSubscription { id status }
    userErrors { field message }
  }
}
```

`prorate: true` refunds the unused portion of the current billing interval.

---

## Staff Members

`staffMembers` lists the humans with admin access to the shop (owner, collaborators, invited staff). Use it for permission audits, assignment dropdowns, or activity attribution.

**Required scope**: `read_users`.

```graphql
query ListStaff($first: Int!, $query: String) {
  staffMembers(first: $first, query: $query) {
    edges {
      node {
        id
        firstName
        lastName
        name
        email
        active
        isShopOwner
        accountType
        locale
        phone
        exists
        initials
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `email:owner@example.com`, `first_name:jamie`, `account_type:collaborator`

**`AccountType` enum**: `REGULAR`, `RESTRICTED`, `COLLABORATOR`, `INVITED`, `REQUESTED`, `SAML`.

Also available: `staffMember(id:)` for a single lookup.

---

## Shop Policies

Shop policies are the merchant's legal pages: refund, privacy, terms of service, shipping, contact, subscription, legal. They're rendered at `/policies/{handle}` on the storefront and linked in checkout.

**Required scope**: `write_legal_policies`.

### Read policies

Policies live on the `Shop` object:

```graphql
query ShopPolicies {
  shop {
    shopPolicies {
      id
      type
      title
      body
      url
      createdAt
      updatedAt
    }
  }
}
```

### Update a policy

```graphql
mutation UpdateShopPolicy($shopPolicy: ShopPolicyInput!) {
  shopPolicyUpdate(shopPolicy: $shopPolicy) {
    shopPolicy { id type body }
    userErrors { field message code }
  }
}
```

**Variables example**:

```json
{
  "shopPolicy": {
    "type": "REFUND_POLICY",
    "body": "<h2>30-Day Returns</h2><p>Unworn items in original packaging may be returned within 30 days of delivery for a full refund...</p>"
  }
}
```

**`ShopPolicyType` enum**: `REFUND_POLICY`, `PRIVACY_POLICY`, `TERMS_OF_SERVICE`, `SHIPPING_POLICY`, `CONTACT_INFORMATION`, `SUBSCRIPTION_POLICY`, `LEGAL_NOTICE`, `TERMS_OF_SALE`.

The `body` is HTML. Setting it to an empty string effectively removes the policy from the storefront.

---

## Events (Audit Log)

`events` is Shopify's built-in activity log: order placed, product updated, refund issued, fulfillment created, etc. Use it for audit trails, change-tracking dashboards, and debugging mysterious state transitions ("when did this product go to DRAFT, and who did it?").

**Required scope**: varies by subject (`read_orders` for order events, `read_products` for product events, etc.).

### List events

```graphql
query ListEvents($first: Int!, $query: String, $sortKey: EventSortKeys) {
  events(first: $first, query: $query, sortKey: $sortKey, reverse: true) {
    edges {
      node {
        id
        createdAt
        message
        criticalAlert
        appTitle
        attributeToApp
        attributeToUser
        ... on BasicEvent {
          action
          subjectId
          subjectType
        }
        ... on CommentEvent {
          author { displayName }
        }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**:
- `subject_type:PRODUCT` — product events only
- `action:create` / `action:destroy` / `action:update`
- `created_at:>2026-04-01`
- `comments:false` — exclude staff comments
- `id:>123456789` — events after a specific event ID (useful for incremental sync)

**Subject types** (common): `PRODUCT`, `PRODUCT_VARIANT`, `COLLECTION`, `ORDER`, `ARTICLE`, `BLOG`, `PAGE`, `CUSTOMER`.

**Sort keys**: `CREATED_AT`, `ID` (default).

### Events on a specific resource

Most resources (Order, Product, Customer, etc.) expose an `events` connection directly — much faster than filtering the global stream:

```graphql
query OrderEvents($id: ID!) {
  order(id: $id) {
    id
    events(first: 50, sortKey: CREATED_AT, reverse: true) {
      edges { node { id createdAt message attributeToUser attributeToApp appTitle } }
    }
  }
}
```

`attributeToUser: true` means a staff member performed the action. `attributeToApp: true` means an app did it (often your own) — useful for separating human from automated changes. Combine with `appTitle` to see which app.

---

## Shopify Payments — Account, Balance, Payouts

If the merchant uses Shopify Payments as their processor, `shopifyPaymentsAccount` exposes the financial back-office: current balance, payout history, disputes, bank accounts.

**Required scope**: `read_shopify_payments_payouts` (or `read_shopify_payments_disputes` / `read_shopify_payments_bank_accounts` for subsets).

### Account overview

```graphql
query ShopifyPaymentsOverview {
  shopifyPaymentsAccount {
    id
    country
    defaultCurrency
    activated
    chargeStatementDescriptor
    balance { amount currencyCode }
    payoutSchedule { interval monthlyAnchor weeklyAnchor }
    bankAccounts(first: 5) {
      edges {
        node {
          id
          status
          bankName
          accountNumberLastDigits
          currency
        }
      }
    }
  }
}
```

`balance` is a list (multi-currency accounts have one entry per currency). A `null` account means the shop isn't enrolled in Shopify Payments.

### Payouts

```graphql
query ListPayouts($first: Int!, $query: String) {
  shopifyPaymentsAccount {
    payouts(first: $first, query: $query, sortKey: ISSUED_AT, reverse: true) {
      edges {
        node {
          id
          issuedAt
          status
          net { amount currencyCode }
          summary {
            chargesGross { amount currencyCode }
            chargesFee { amount currencyCode }
            refundsFeeGross { amount currencyCode }
            adjustmentsGross { amount currencyCode }
            reservedFundsGross { amount currencyCode }
          }
          bankAccount { id bankName accountNumberLastDigits }
          transactionType
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

> ⚠️ **`payout.gross` is deprecated** — read `payout.net` instead. Use the `summary` breakdown for gross/fee detail.

**Filter examples**: `issued_at:>2026-01-01`, `status:paid`, `status:failed`

**`ShopifyPaymentsPayoutStatus`** — current values are `SCHEDULED` (assigned but not yet sent to the bank), `PAID` (deposited), `FAILED` (bank declined), and `CANCELED` (Shopify canceled). `IN_TRANSIT` was in older versions but is now **deprecated** — don't rely on it for new code.

### Disputes (chargebacks & inquiries)

```graphql
query ListDisputes($first: Int!) {
  shopifyPaymentsAccount {
    disputes(first: $first) {
      edges {
        node {
          id
          status
          reasonDetails { reason networkReasonCode }
          amount { amount currencyCode }
          initiatedAt
          evidenceDueBy
          finalizedOn
          order { id name }
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

Use `evidenceDueBy` to drive a "disputes to respond to this week" dashboard. Disputes past their `evidenceDueBy` without a response typically lose by default.

### Balance transactions

`balanceTransactions` is the ledger: every charge, refund, fee, adjustment, and payout that moved money in the account. Use it to reconcile Shopify Payments totals against the merchant's bank statement.

```graphql
query BalanceTransactions($first: Int!, $payoutId: ID) {
  shopifyPaymentsAccount {
    balanceTransactions(first: $first, payoutId: $payoutId) {
      edges {
        node {
          id
          type
          net { amount currencyCode }
          fee { amount currencyCode }
          amount { amount currencyCode }
          transactionDate
          payoutId
          payoutStatus
          sourceOrderTransactionId
          sourceOrder { id name }
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

Pass a `payoutId` to list only transactions that settled in one specific payout — the standard "what made up this deposit" reconciliation query.

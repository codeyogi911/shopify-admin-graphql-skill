# Marketing — Segments, Selling Plans, Gift Cards, Marketing Activities

Patterns for customer targeting, subscription/try-before-you-buy offers, gift-card lifecycle, and tracking external campaigns.

---

## Customer Segments

A `Segment` is a dynamic customer group defined by a ShopifyQL-flavored query. Membership recomputes as customer attributes change — you don't write members in; you write the rule.

**Required scope**: `write_customers`.

### List segments

```graphql
query ListSegments($first: Int!, $query: String) {
  segments(first: $first, query: $query) {
    edges {
      node {
        id
        name
        query
        creationDate
        lastEditDate
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `name:vip`, `id:123`

**Sort keys**: `ID`, `CREATION_DATE`, `LAST_EDIT_DATE`, `UPDATE_DATE`, `RELEVANCE`

### Create a segment

```graphql
mutation CreateSegment($name: String!, $query: String!) {
  segmentCreate(name: $name, query: $query) {
    segment {
      id
      name
      query
    }
    userErrors { field message }
  }
}
```

**Variables example** (subscribed customers who've spent over $500):

```json
{
  "name": "VIP Subscribers",
  "query": "email_subscription_status = 'SUBSCRIBED' AND amount_spent > 500.0"
}
```

### Segment query syntax — common facts

The `query` field uses a dedicated segment DSL (not the generic search-syntax `field:value` format). Typical facts you can assert on:

| Fact | Example |
|---|---|
| `email_subscription_status` | `= 'SUBSCRIBED'` / `= 'NOT_SUBSCRIBED'` |
| `sms_subscription_status` | `= 'SUBSCRIBED'` |
| `amount_spent` | `> 500.0` |
| `number_of_orders` | `>= 3` |
| `last_order_date` | `> '2026-01-01'` |
| `customer_tags` | `CONTAINS 'vip'` |
| `customer_cities` | `CONTAINS 'Austin'` |
| `customer_countries` | `CONTAINS 'US'` |
| `last_abandoned_checkout_date` | `> '2026-03-01'` |
| `predicted_spend_tier` | `= 'HIGH'` |

Combine with `AND`, `OR`, `NOT`, parentheses.

### List members of a segment

```graphql
query SegmentMembers($segmentId: ID!, $first: Int!, $after: String) {
  customerSegmentMembers(segmentId: $segmentId, first: $first, after: $after) {
    totalCount
    edges {
      node {
        id
        firstName
        lastName
        displayName
        defaultEmailAddress { emailAddress }
        numberOfOrders
        amountSpent { amount currencyCode }
      }
    }
    pageInfo { hasNextPage endCursor }
    statistics { }
  }
}
```

Max page size is **1000**. Use `sortKey: AMOUNT_SPENT` / `LAST_ORDER_DATE` / `NUMBER_OF_ORDERS` for ranked exports.

---

## Selling Plan Groups (Subscriptions, Try-Before-You-Buy, Pre-Orders)

A `SellingPlanGroup` bundles one or more `SellingPlan`s that offer alternative purchase terms on a product — monthly subscriptions, 14-day trials, pre-orders, etc. You create the group + plans together, then attach products to it.

**Required scopes**: `write_products` + `write_purchase_options` (or `write_own_subscription_contracts`).

### Create a selling plan group with subscription plans

```graphql
mutation CreateSellingPlanGroup($input: SellingPlanGroupInput!, $resources: SellingPlanGroupResourceInput) {
  sellingPlanGroupCreate(input: $input, resources: $resources) {
    sellingPlanGroup {
      id
      name
      merchantCode
      sellingPlans(first: 10) {
        edges { node { id name category options } }
      }
    }
    userErrors { field message code }
  }
}
```

**Variables example** (subscribe-and-save, 10% off, ship every 30 days):

```json
{
  "input": {
    "name": "Subscribe & Save",
    "merchantCode": "subscribe-and-save-10",
    "options": ["Delivery every"],
    "sellingPlansToCreate": [
      {
        "name": "Deliver every 30 days",
        "category": "SUBSCRIPTION",
        "options": ["30 days"],
        "billingPolicy": {
          "recurring": { "interval": "DAY", "intervalCount": 30 }
        },
        "deliveryPolicy": {
          "recurring": { "interval": "DAY", "intervalCount": 30, "preAnchorBehavior": "ASAP", "intent": "FULFILLMENT_BEGIN" }
        },
        "pricingPolicies": [
          {
            "fixed": {
              "adjustmentType": "PERCENTAGE",
              "adjustmentValue": { "percentage": 10.0 }
            }
          }
        ]
      }
    ]
  },
  "resources": {
    "productIds": ["gid://shopify/Product/123"]
  }
}
```

**`SellingPlanCategory` enum**: `SUBSCRIPTION`, `PRE_ORDER`, `TRY_BEFORE_YOU_BUY`, `OTHER`.

**Billing/delivery interval units**: `DAY`, `WEEK`, `MONTH`, `YEAR`.

**Pricing policy adjustment types**: `PERCENTAGE`, `FIXED_AMOUNT`, `PRICE` (hard-set price).

### Attach existing products to a selling plan group

```graphql
mutation AddProductsToSellingPlanGroup($id: ID!, $productIds: [ID!]!) {
  sellingPlanGroupAddProducts(id: $id, productIds: $productIds) {
    sellingPlanGroup { id productCount }
    userErrors { field message code }
  }
}
```

To attach individual variants instead of whole products, use `sellingPlanGroupAddProductVariants`. Reverse with `sellingPlanGroupRemoveProducts` / `sellingPlanGroupRemoveProductVariants`.

---

## Gift Cards

Gift cards are a stored-value balance the customer can apply at checkout. Shopify generates a random 16-character code unless you supply one.

**Required scope**: `write_gift_cards`.

### Create a gift card

```graphql
mutation CreateGiftCard($input: GiftCardCreateInput!) {
  giftCardCreate(input: $input) {
    giftCard {
      id
      maskedCode
      initialValue { amount currencyCode }
      balance { amount currencyCode }
      expiresOn
      enabled
      customer { id displayName email }
    }
    giftCardCode
    userErrors { field message code }
  }
}
```

**Variables example** (issue a $50 card to an existing customer, expires in 1 year):

```json
{
  "input": {
    "initialValue": "50.00",
    "customerId": "gid://shopify/Customer/123",
    "expiresOn": "2027-04-11",
    "note": "Holiday promo 2026 winner",
    "recipientAttributes": {
      "recipientPersonalMessage": "Enjoy! — Acme Team",
      "sendNotificationAt": "2026-04-11T09:00:00Z"
    }
  }
}
```

Two reads of the code you must capture:
- **`giftCardCode`** (top-level payload field) — the **full, plaintext code**, returned **once** at creation time. Save it now; it can't be retrieved later.
- **`giftCard.maskedCode`** — last 4 digits only. Safe to show in dashboards.

### Update balance / note / expiry

```graphql
mutation UpdateGiftCard($id: ID!, $input: GiftCardUpdateInput!) {
  giftCardUpdate(id: $id, input: $input) {
    giftCard { id balance { amount currencyCode } expiresOn note }
    userErrors { field message code }
  }
}
```

### Deactivate (permanent)

```graphql
mutation DeactivateGiftCard($id: ID!) {
  giftCardDeactivate(id: $id) {
    giftCard { id enabled }
    userErrors { field message }
  }
}
```

> Deactivation is **permanent** — a deactivated gift card cannot be re-enabled. Use this for lost/compromised cards, not temporary holds.

### List gift cards

```graphql
query ListGiftCards($first: Int!, $query: String) {
  giftCards(first: $first, query: $query) {
    edges {
      node {
        id
        maskedCode
        initialValue { amount currencyCode }
        balance { amount currencyCode }
        enabled
        expiresOn
        customer { id email }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `customer_id:123`, `status:enabled` / `status:disabled`, `balance:>0`, `expires_on:<2026-12-31`

---

## Marketing Activities (External Campaigns)

If you run campaigns outside Shopify (Meta Ads, Google Ads, an ESP, a headless site), register them as **external marketing activities** so their impressions, clicks, and revenue show up inside Shopify's marketing reports alongside native campaigns.

**Required scope**: `write_marketing_events`.

The flow is: register the activity once → push engagement metrics on a schedule.

### 1. Register the external activity

```graphql
mutation CreateExternalMarketingActivity($input: MarketingActivityCreateExternalInput!) {
  marketingActivityCreateExternal(input: $input) {
    marketingActivity {
      id
      title
      status
      marketingChannelType
      tactic
    }
    userErrors { field message code }
  }
}
```

**Variables example**:

```json
{
  "input": {
    "remoteId": "meta-campaign-42",
    "title": "Spring Sale — Meta Prospecting",
    "remoteUrl": "https://business.facebook.com/adsmanager/manage/campaigns?act=...&selected_campaign_ids=42",
    "remotePreviewImageUrl": "https://example.com/creatives/spring-sale-v1.jpg",
    "marketingChannelType": "SOCIAL",
    "tactic": "AD",
    "utm": {
      "source": "facebook",
      "medium": "paid_social",
      "campaign": "spring-sale-2026"
    },
    "status": "ACTIVE"
  }
}
```

**`MarketingChannelType` enum**: `SEARCH`, `DISPLAY`, `SOCIAL`, `EMAIL`, `REFERRAL`, `DIRECT`.

**`MarketingTactic` enum** (common): `AD`, `AFFILIATE`, `LINK`, `LOYALTY`, `MESSAGE`, `NEWSLETTER`, `NOTIFICATION`, `POST`, `RETARGETING`, `SEARCH`, `SEO`, `DISPLAY`, `TRANSACTIONAL`, `STOREFRONT_APP`.

### 2. Push engagement metrics

```graphql
mutation RecordEngagement($marketingActivityId: ID!, $marketingEngagement: MarketingEngagementInput!) {
  marketingEngagementCreate(
    marketingActivityId: $marketingActivityId,
    marketingEngagement: $marketingEngagement
  ) {
    marketingEngagement {
      occurredOn
      impressionsCount
      clicksCount
      sales { amount currencyCode }
      adSpend { amount currencyCode }
    }
    userErrors { field message }
  }
}
```

**Variables example** (daily rollup):

```json
{
  "marketingActivityId": "gid://shopify/MarketingActivity/99",
  "marketingEngagement": {
    "occurredOn": "2026-04-10",
    "utcOffset": "-07:00",
    "impressionsCount": 12400,
    "clicksCount": 312,
    "adSpend": { "amount": "148.50", "currencyCode": "USD" },
    "sales": { "amount": "2210.75", "currencyCode": "USD" },
    "isCumulative": false
  }
}
```

**`isCumulative`** controls the semantics:
- `false` (incremental) — this record holds **only the deltas** for `occurredOn`. Shopify sums daily rows to get a total.
- `true` (cumulative) — this record holds **all-time totals** through `occurredOn`. Shopify diffs consecutive rows to infer daily deltas.

Pick one mode and stick with it per activity. Mixing cumulative and incremental rows gives garbage totals.

You can also push engagement to a channel-level feed instead of a single activity by passing `channelHandle` in place of `marketingActivityId`.

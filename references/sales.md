# Sales — Orders, Discounts, Fulfillment

Patterns for reading orders, creating discounts, and fulfilling shipments.

---

## Orders

### List orders (paginated)

```graphql
query ListOrders($first: Int!, $after: String, $query: String) {
  orders(first: $first, after: $after, query: $query) {
    edges {
      node {
        id
        name
        createdAt
        displayFinancialStatus
        displayFulfillmentStatus
        totalPriceSet { shopMoney { amount currencyCode } }
        customer { id displayName email }
        lineItems(first: 10) {
          edges {
            node {
              title
              quantity
              originalUnitPriceSet { shopMoney { amount currencyCode } }
              sku
            }
          }
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Filter examples** (`query` variable):
- `financial_status:paid` — paid orders
- `financial_status:authorized` — authorized payments
- `fulfillment_status:unfulfilled` — unfulfilled
- `status:open` / `status:closed` / `status:cancelled`
- `created_at:>2024-01-01` — after date
- `updated_at:>2024-06-01` — recently updated
- `customer_id:123` — specific customer
- `tag:rush` — tagged orders
- `email:john@example.com` — by email
- `sku:ABC-123` — by line item SKU
- `risk_level:high` — fraud risk
- `return_status:in_progress` — active returns
- `test:true` — test orders only

**Sort keys**: `CREATED_AT`, `CUSTOMER_NAME`, `FINANCIAL_STATUS`, `FULFILLMENT_STATUS`, `ORDER_NUMBER`, `PROCESSED_AT`, `TOTAL_PRICE`, `UPDATED_AT`, `RELEVANCE`, `ID`

### Get single order

```graphql
query GetOrder($id: ID!) {
  order(id: $id) {
    id
    name
    createdAt
    processedAt
    displayFinancialStatus
    displayFulfillmentStatus
    totalPriceSet { shopMoney { amount currencyCode } }
    subtotalPriceSet { shopMoney { amount currencyCode } }
    totalShippingPriceSet { shopMoney { amount currencyCode } }
    totalTaxSet { shopMoney { amount currencyCode } }
    customer { id displayName email phone }
    shippingAddress { address1 address2 city province zip country }
    billingAddress { address1 address2 city province zip country }
    lineItems(first: 50) {
      edges {
        node {
          id
          title
          quantity
          sku
          originalUnitPriceSet { shopMoney { amount currencyCode } }
          variant { id title }
        }
      }
    }
    fulfillments { id status trackingInfo { number url company } }
    transactions { id kind status amountSet { shopMoney { amount currencyCode } } }
    note
    tags
  }
}
```

---

## Discounts

### Create basic discount code

```graphql
mutation CreateDiscountCode($basicCodeDiscount: DiscountCodeBasicInput!) {
  discountCodeBasicCreate(basicCodeDiscount: $basicCodeDiscount) {
    codeDiscountNode {
      id
      codeDiscount {
        ... on DiscountCodeBasic {
          title
          codes(first: 1) { edges { node { code } } }
          startsAt
          endsAt
        }
      }
    }
    userErrors { field message code }
  }
}
```

---

## Fulfillment

### Create fulfillment

```graphql
mutation FulfillOrder($fulfillment: FulfillmentInput!) {
  fulfillmentCreate(fulfillment: $fulfillment) {
    fulfillment {
      id
      status
      trackingInfo { number url company }
    }
    userErrors { field message }
  }
}
```

---

## Draft Orders

Draft orders cover phone/in-person sales, invoices, wholesale pricing, and pre-orders. They convert to real orders via `draftOrderComplete`.

**Required scope**: `write_draft_orders` (or `write_quick_sale`).

### List draft orders

```graphql
query ListDraftOrders($first: Int!, $after: String, $query: String) {
  draftOrders(first: $first, after: $after, query: $query) {
    edges {
      node {
        id
        name
        status
        totalPriceSet { shopMoney { amount currencyCode } }
        customer { id displayName email }
        createdAt
        updatedAt
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `status:OPEN`, `customer_id:123`, `tag:wholesale`, `created_at:>2026-01-01`, `updated_at:>2026-03-01`

### Create draft order

```graphql
mutation CreateDraftOrder($input: DraftOrderInput!) {
  draftOrderCreate(input: $input) {
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

**Variables example**:

```json
{
  "input": {
    "email": "buyer@example.com",
    "lineItems": [
      { "variantId": "gid://shopify/ProductVariant/123", "quantity": 2 },
      { "title": "Custom Engraving", "originalUnitPrice": "15.00", "quantity": 1 }
    ],
    "appliedDiscount": { "value": 10.0, "valueType": "PERCENTAGE", "title": "Wholesale" },
    "shippingAddress": { "address1": "1 Main St", "city": "Austin", "province": "TX", "zip": "78701", "country": "United States" },
    "tags": ["wholesale"],
    "note": "Net-30 invoice"
  }
}
```

Line items can reference existing variants via `variantId` or be fully custom (set `title` + `originalUnitPrice`, omit `variantId`).

### Complete draft order (convert to real order)

```graphql
mutation CompleteDraftOrder($id: ID!, $paymentGatewayId: ID, $sourceName: String) {
  draftOrderComplete(id: $id, paymentGatewayId: $paymentGatewayId, sourceName: $sourceName) {
    draftOrder {
      id
      order { id name }
    }
    userErrors { field message }
  }
}
```

Note: the legacy `paymentPending` argument is deprecated. For deferred payment, use **payment terms** on the draft order instead.

### Delete draft order

```graphql
mutation DeleteDraftOrder($input: DraftOrderDeleteInput!) {
  draftOrderDelete(input: $input) {
    deletedId
    userErrors { field message }
  }
}
```

---

## Returns

Returns track a buyer's intent to ship items back. Two entry points:

- **`returnCreate`** — return goes straight to `OPEN` (skip customer-initiated approval). Use when the merchant/app is authoritative.
- **`returnRequest`** — return starts in `REQUESTED` state; merchant calls `returnApproveRequest` or `returnDeclineRequest` afterwards. Use for customer-initiated flows.

**Required scope**: `write_returns` (or `write_marketplace_returns`).

### Create a return (skip approval)

```graphql
mutation CreateReturn($returnInput: ReturnInput!) {
  returnCreate(returnInput: $returnInput) {
    return {
      id
      name
      status
      totalQuantity
      order { id name }
    }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "returnInput": {
    "orderId": "gid://shopify/Order/123",
    "returnLineItems": [
      {
        "fulfillmentLineItemId": "gid://shopify/FulfillmentLineItem/456",
        "quantity": 1,
        "returnReason": "SIZE_TOO_SMALL",
        "returnReasonNote": "Customer wanted a medium"
      }
    ]
  }
}
```

**ReturnReason enum** (common values): `SIZE_TOO_SMALL`, `SIZE_TOO_LARGE`, `UNWANTED`, `NOT_AS_DESCRIBED`, `WRONG_ITEM`, `DEFECTIVE`, `STYLE`, `COLOR`, `OTHER`.

### Request a return (customer-initiated)

```graphql
mutation RequestReturn($input: ReturnRequestInput!) {
  returnRequest(input: $input) {
    return { id status }
    userErrors { field message }
  }
}
```

Then approve or decline:

```graphql
mutation ApproveReturnRequest($input: ReturnApproveRequestInput!) {
  returnApproveRequest(input: $input) {
    return { id status }
    userErrors { field message }
  }
}
```

### Query returns on an order

```graphql
query OrderReturns($id: ID!) {
  order(id: $id) {
    id
    returns(first: 10) {
      edges {
        node {
          id
          name
          status
          totalQuantity
          returnLineItems(first: 20) {
            edges {
              node {
                ... on ReturnLineItem {
                  id
                  quantity
                  returnReason
                }
              }
            }
          }
          refunds(first: 5) { edges { node { id totalRefundedSet { shopMoney { amount currencyCode } } } } }
        }
      }
    }
  }
}
```

---

## Refunds

Refunds move money back to the customer and (optionally) restock inventory. A refund can be tied to a return or stand alone.

**Required scope**: `write_orders` (or `write_marketplace_orders` / `write_buyer_membership_orders`).

> **Idempotency required (API ≥ 2026-04)**: `refundCreate` must be called with the `@idempotent` directive on the mutation field, passing an idempotency key. Reuse the same key across retries to guarantee a single refund per logical operation.

### Create refund

```graphql
mutation CreateRefund($input: RefundInput!, $idempotencyKey: String!) {
  refundCreate(input: $input) @idempotent(key: $idempotencyKey) {
    refund {
      id
      totalRefundedSet { shopMoney { amount currencyCode } }
      order { id name }
    }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "key": "refund-order-123-attempt-1",
  "input": {
    "orderId": "gid://shopify/Order/123",
    "note": "Customer return — damaged in transit",
    "notify": true,
    "shipping": { "fullRefund": true },
    "refundLineItems": [
      {
        "lineItemId": "gid://shopify/LineItem/789",
        "quantity": 1,
        "restockType": "RETURN",
        "locationId": "gid://shopify/Location/456"
      }
    ],
    "transactions": [
      {
        "orderId": "gid://shopify/Order/123",
        "parentId": "gid://shopify/OrderTransaction/999",
        "amount": "29.99",
        "gateway": "shopify_payments",
        "kind": "REFUND"
      }
    ]
  }
}
```

`restockType` enum: `NO_RESTOCK`, `CANCEL`, `RETURN`, `LEGACY_RESTOCK`.

---

## Order Editing

Edit a placed order's line items, quantities, or discounts. Flow: **begin → modify → commit**. Changes stay in a transactional session until committed.

**Required scope**: `write_order_edits`.

### 1. Begin an edit session

```graphql
mutation BeginOrderEdit($id: ID!) {
  orderEditBegin(id: $id) {
    calculatedOrder {
      id
      lineItems(first: 50) {
        edges { node { id title quantity editableQuantity } }
      }
    }
    userErrors { field message }
  }
}
```

Hold onto `calculatedOrder.id` — you pass it to every subsequent mutation in the session.

### 2. Stage modifications

```graphql
mutation AddVariantToOrder($id: ID!, $variantId: ID!, $quantity: Int!) {
  orderEditAddVariant(id: $id, variantId: $variantId, quantity: $quantity) {
    calculatedOrder { id addedLineItems(first: 10) { edges { node { id quantity } } } }
    userErrors { field message }
  }
}

mutation SetOrderLineItemQuantity($id: ID!, $lineItemId: ID!, $quantity: Int!) {
  orderEditSetQuantity(id: $id, lineItemId: $lineItemId, quantity: $quantity) {
    calculatedOrder { id }
    userErrors { field message }
  }
}
```

Other stage mutations: `orderEditAddCustomItem`, `orderEditAddLineItemDiscount`, `orderEditRemoveLineItemDiscount`.

### 3. Commit the edit

```graphql
mutation CommitOrderEdit($id: ID!, $notifyCustomer: Boolean, $staffNote: String) {
  orderEditCommit(id: $id, notifyCustomer: $notifyCustomer, staffNote: $staffNote) {
    order { id name }
    userErrors { field message }
  }
}
```

Passing `notifyCustomer: true` sends the customer an updated order confirmation.

---

## Order Risk

`Order.riskLevel` is deprecated. Use `Order.risk` (returns `OrderRiskSummary`).

```graphql
query OrderRisk($id: ID!) {
  order(id: $id) {
    id
    name
    risk {
      recommendation
      assessments {
        riskLevel
        provider { title }
        facts { description sentiment }
      }
    }
  }
}
```

**`OrderRiskRecommendationResult` enum**: `ACCEPT`, `INVESTIGATE`, `CANCEL`, `NONE`.

Combine with the `risk_level:high` order filter to triage suspicious orders in bulk.

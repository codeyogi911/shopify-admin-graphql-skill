# Shopify Admin GraphQL — Common Patterns Reference

Embedded patterns for the most common store management operations.
Agents can extend this file by running introspection queries and appending new sections.

---

## Products

### List products (paginated)

```graphql
query ListProducts($first: Int!, $after: String, $query: String) {
  products(first: $first, after: $after, query: $query) {
    edges {
      node {
        id
        title
        handle
        status
        vendor
        productType
        tags
        totalInventory
        createdAt
        updatedAt
        variants(first: 10) {
          edges {
            node {
              id
              title
              sku
              price
              inventoryQuantity
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
- `title:*hoodie*` — title contains "hoodie"
- `status:ACTIVE` — only active products
- `vendor:Nike` — by vendor
- `product_type:shoes` — by type
- `created_at:>2024-01-01` — created after date
- `inventory_total:0` — out of stock
- `tag:sale` — has tag
- `sku:ABC-123` — by variant SKU

**Sort keys**: `TITLE`, `PRODUCT_TYPE`, `VENDOR`, `INVENTORY_TOTAL`, `UPDATED_AT`, `CREATED_AT`, `PUBLISHED_AT`, `RELEVANCE`, `ID`

### Get single product

```graphql
query GetProduct($id: ID!) {
  product(id: $id) {
    id
    title
    descriptionHtml
    handle
    status
    vendor
    productType
    tags
    options {
      id
      name
      optionValues { id name }
    }
    variants(first: 50) {
      edges {
        node {
          id
          title
          sku
          price
          compareAtPrice
          inventoryQuantity
          selectedOptions { name value }
        }
      }
    }
    media(first: 10) {
      edges {
        node {
          ... on MediaImage {
            id
            image { url altText width height }
          }
        }
      }
    }
    seo { title description }
  }
}
```

### Create product

```graphql
mutation CreateProduct($product: ProductCreateInput!, $media: [CreateMediaInput!]) {
  productCreate(product: $product, media: $media) {
    product {
      id
      title
      handle
      options { id name optionValues { id name } }
    }
    userErrors { field message }
  }
}
```

**Variables example**:
```json
{
  "product": {
    "title": "Cool Socks",
    "vendor": "Acme",
    "productType": "Accessories",
    "tags": ["new", "sale"],
    "productOptions": [
      { "name": "Color", "values": [{ "name": "Red" }, { "name": "Blue" }] },
      { "name": "Size", "values": [{ "name": "S" }, { "name": "M" }, { "name": "L" }] }
    ]
  }
}
```

Note: `productCreate` only creates the initial variant. Use `productVariantsBulkCreate` for multiple variants.

### Update product

```graphql
mutation UpdateProduct($input: ProductInput!) {
  productUpdate(input: $input) {
    product { id title }
    userErrors { field message }
  }
}
```

### Delete product

```graphql
mutation DeleteProduct($input: ProductDeleteInput!) {
  productDelete(input: $input) {
    deletedProductId
    userErrors { field message }
  }
}
```

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

## Customers

### List customers

```graphql
query ListCustomers($first: Int!, $after: String, $query: String) {
  customers(first: $first, after: $after, query: $query) {
    edges {
      node {
        id
        displayName
        email
        phone
        numberOfOrders
        amountSpent { amount currencyCode }
        createdAt
        tags
        defaultAddress { address1 city province zip country }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `email:john@example.com`, `tag:vip`, `country:US`, `orders_count:>5`, `total_spent:>100`

### Create customer

```graphql
mutation CreateCustomer($input: CustomerInput!) {
  customerCreate(input: $input) {
    customer { id displayName email }
    userErrors { field message }
  }
}
```

### Update customer

```graphql
mutation UpdateCustomer($input: CustomerInput!) {
  customerUpdate(input: $input) {
    customer { id displayName email }
    userErrors { field message }
  }
}
```

---

## Inventory

### Get inventory levels for a location

```graphql
query InventoryLevels($locationId: ID!, $first: Int!) {
  location(id: $locationId) {
    name
    inventoryLevels(first: $first) {
      edges {
        node {
          id
          quantities(names: ["available", "incoming", "committed", "on_hand"]) {
            name
            quantity
          }
          item {
            id
            sku
            variant { id title product { id title } }
          }
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

### Adjust inventory quantities

```graphql
mutation AdjustInventory($input: InventoryAdjustQuantitiesInput!) {
  inventoryAdjustQuantities(input: $input) {
    inventoryAdjustmentGroup {
      reason
      changes {
        name
        delta
        quantityAfterChange
      }
    }
    userErrors { field message }
  }
}
```

**Variables example**:
```json
{
  "input": {
    "reason": "correction",
    "name": "available",
    "changes": [
      {
        "delta": 10,
        "inventoryItemId": "gid://shopify/InventoryItem/123",
        "locationId": "gid://shopify/Location/456"
      }
    ]
  }
}
```

---

## Collections

### List collections

```graphql
query ListCollections($first: Int!) {
  collections(first: $first) {
    edges {
      node {
        id
        title
        handle
        productsCount { count }
        updatedAt
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

### Create collection

```graphql
mutation CreateCollection($input: CollectionInput!) {
  collectionCreate(input: $input) {
    collection { id title handle }
    userErrors { field message }
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

## Metafields

### Set metafields

```graphql
mutation SetMetafields($metafields: [MetafieldsSetInput!]!) {
  metafieldsSet(metafields: $metafields) {
    metafields { id namespace key value type }
    userErrors { field message }
  }
}
```

**Variables example**:
```json
{
  "metafields": [
    {
      "ownerId": "gid://shopify/Product/123",
      "namespace": "custom",
      "key": "care_instructions",
      "value": "Machine wash cold",
      "type": "single_line_text_field"
    }
  ]
}
```

### Common metafield types

`single_line_text_field`, `multi_line_text_field`, `number_integer`, `number_decimal`, `boolean`, `date`, `date_time`, `url`, `json`, `color`, `weight`, `dimension`, `rating`, `money`, `rich_text_field`, `list.single_line_text_field`

---

## Shop Info

```graphql
query ShopInfo {
  shop {
    name
    email
    myshopifyDomain
    primaryDomain { url host }
    plan { displayName }
    currencyCode
    billingAddress { address1 city province zip country }
    timezoneAbbreviation
  }
}
```

---

## Locations

```graphql
query ListLocations {
  locations(first: 50) {
    edges {
      node {
        id
        name
        isActive
        address { address1 city province zip country }
      }
    }
  }
}
```

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

*To add new patterns: run a targeted introspection query, then append the results below.*

# Analytics & Reporting

ShopifyQL is Shopify's SQL-flavored query language for store analytics. Run it through the `shopifyqlQuery` GraphQL query to get tabular results across sales, inventory, customer, and product data.

**Required scopes**: `read_reports` + Level 2 access to protected customer data (for queries touching name/email/phone/address).

---

## shopifyqlQuery

```graphql
query RunShopifyQL($query: String!) {
  shopifyqlQuery(query: $query) {
    __typename
    ... on TableResponse {
      tableData {
        columns { name dataType displayName }
        rowData
        unformattedData
      }
    }
    ... on ParseError {
      parseErrors { code message range { start { line character } end { line character } } }
    }
  }
}
```

The response is a union (`ShopifyqlResponse`). Always use an inline fragment on both `TableResponse` and `ParseError`.

`tableData.rowData` is a 2D array of strings — each inner array is one row, ordered to match `columns`. Use `unformattedData` when you need raw values for arithmetic (formatted strings include currency symbols, thousand separators, etc.).

---

## ShopifyQL basics

A ShopifyQL query looks like:

```
FROM <dataset>
SHOW <columns>
[GROUP BY <dimensions>]
[WHERE <filter>]
[SINCE <date>] [UNTIL <date>]
[ORDER BY <column> [ASC|DESC]]
[LIMIT <n>]
[WITH TOTALS]
```

### Common datasets

| Dataset | What it holds |
|---|---|
| `sales` | Orders, line items, refunds, taxes, net/gross sales |
| `orders` | Order-level metrics (count, AOV, status) |
| `products` | Product/variant performance, inventory turnover |
| `customers` | New vs returning, LTV, cohorts |
| `inventory` | Stock on hand, sell-through, days of supply |

### Date shortcuts

`SINCE -7d`, `SINCE -3m`, `SINCE -1y`, `SINCE 'yyyy-mm-dd'`, `UNTIL today`.

---

## Examples

### Monthly sales for the last 3 months

```json
{
  "query": "FROM sales SHOW total_sales GROUP BY month SINCE -3m ORDER BY month WITH TOTALS"
}
```

### Top 10 products by gross sales last 30 days

```json
{
  "query": "FROM sales SHOW gross_sales GROUP BY product_title SINCE -30d ORDER BY gross_sales DESC LIMIT 10"
}
```

### New vs returning customer revenue this quarter

```json
{
  "query": "FROM sales SHOW net_sales GROUP BY customer_type SINCE -3m"
}
```

### Units sold by variant SKU

```json
{
  "query": "FROM sales SHOW net_quantity GROUP BY variant_sku SINCE -7d ORDER BY net_quantity DESC LIMIT 25"
}
```

### Order count and AOV by day

```json
{
  "query": "FROM orders SHOW orders, average_order_value GROUP BY day SINCE -14d ORDER BY day"
}
```

---

## Handling parse errors

`shopifyqlQuery` does not throw on malformed queries — it returns a `ParseError` payload. Always branch on `__typename`:

```typescript
if (result.shopifyqlQuery.__typename === "ParseError") {
  // surface result.shopifyqlQuery.parseErrors to the caller
} else {
  // consume result.shopifyqlQuery.tableData
}
```

---

## When to use this vs Bulk Operations

| Use `shopifyqlQuery` when | Use Bulk Operations when |
|---|---|
| You want aggregates (sums, counts, GROUP BY) | You need raw rows for every matching record |
| You're building a dashboard or report | You're exporting/syncing to a data warehouse |
| Data volume is small (table fits in one response) | Data volume is large (MBs to GBs, JSONL output) |
| You want sub-second latency | You can wait minutes for completion |

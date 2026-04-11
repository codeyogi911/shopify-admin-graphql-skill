# Customers

Patterns for listing, creating, and updating customer records.

---

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

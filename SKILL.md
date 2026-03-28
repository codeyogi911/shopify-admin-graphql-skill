---
name: shopify-admin-graphql
description: >
  Generate correct Shopify Admin GraphQL queries and mutations for store
  management — products, orders, customers, inventory, collections, discounts,
  fulfillment, metafields, and more. Includes an introspection workflow to
  discover schema at runtime and embedded common patterns. Use when the user
  asks about Shopify Admin API, GraphQL operations, store data, or any
  Shopify e-commerce integration. Triggers on: "Shopify", "Admin API",
  "GraphQL", "Shopify query", "Shopify mutation", "products API", "orders API",
  "customers API", "inventory API", "Shopify schema", "store management".
---

# Shopify Admin GraphQL

Generate and validate GraphQL operations against the Shopify Admin API.
For the full schema reference, see the [common patterns](reference.md).

## API Basics

- **Endpoint**: `POST https://{shop}.myshopify.com/admin/api/2026-01/graphql.json`
- **Auth header**: `X-Shopify-Access-Token: {token}`
- **Latest stable version**: `2026-01`
- **All connections use cursor-based pagination** (`first`/`after`, `last`/`before`)
- **Mutations return `userErrors`** — always request them; a 200 status does not mean success

## Schema Discovery via Introspection

When you need to discover fields, arguments, or types not covered in [reference.md](reference.md), run a GraphQL introspection query against the store.

### Full schema introspection (heavy — use sparingly)

```graphql
query IntrospectSchema {
  __schema {
    queryType { name }
    mutationType { name }
    types {
      name
      kind
      description
      fields {
        name
        type { name kind ofType { name kind } }
        args { name type { name kind ofType { name kind } } }
      }
      inputFields {
        name
        type { name kind ofType { name kind } }
      }
      enumValues { name }
    }
  }
}
```

### Targeted type introspection (preferred — lightweight)

```graphql
query IntrospectType($typeName: String!) {
  __type(name: $typeName) {
    name
    kind
    description
    fields {
      name
      description
      type { name kind ofType { name kind ofType { name kind } } }
      args { name type { name kind ofType { name kind } } }
    }
    inputFields {
      name
      description
      type { name kind ofType { name kind ofType { name kind } } }
    }
    enumValues { name description }
  }
}
```

Use with variables like `{"typeName": "Product"}`, `{"typeName": "ProductCreateInput"}`, etc.

### Discover available queries and mutations

```graphql
query ListQueries {
  __type(name: "QueryRoot") {
    fields { name description args { name type { name kind } } }
  }
}

query ListMutations {
  __type(name: "Mutation") {
    fields { name description args { name type { name kind } } }
  }
}
```

### Workflow: update the skill with new patterns

After running introspection and discovering useful patterns:

1. Read `reference.md` in this skill directory
2. Append the new patterns (queries, mutations, input types) to the appropriate section
3. Keep the format consistent with existing entries

## Code Generation Rules

1. **Always use variables** for dynamic values — never inline IDs or user input into query strings.
2. **Always request `userErrors { field message }`** in every mutation.
3. **Always include `pageInfo { hasNextPage endCursor }`** in paginated queries.
4. **Use `first: 50` as a sensible default** for list queries (max 250).
5. **Request only the fields you need** — Shopify rate-limits by calculated query cost.
6. **Use `query` argument for server-side filtering** instead of client-side filtering.
7. **Prefer `productSet` over `productUpdate`** for complex multi-variant operations.
8. **Global IDs**: Shopify uses GIDs like `gid://shopify/Product/123`. Always use the full GID.

## Authentication Patterns

### cURL

```bash
curl -X POST \
  https://{shop}.myshopify.com/admin/api/2026-01/graphql.json \
  -H 'Content-Type: application/json' \
  -H 'X-Shopify-Access-Token: {access_token}' \
  -d '{"query": "...", "variables": {}}'
```

### Node.js (@shopify/shopify-api)

```typescript
const client = new shopify.clients.Graphql({ session });
const response = await client.query({
  data: { query: QUERY_STRING, variables: { /* ... */ } },
});
```

### Direct API Access (embedded apps)

```typescript
const response = await fetch('shopify:admin/api/2026-01/graphql.json', {
  method: 'POST',
  body: JSON.stringify({ query: QUERY_STRING, variables: {} }),
});
const { data } = await response.json();
```

## Rate Limits

- **Bucket**: 1000 cost points, restores at 50/second
- **Single query max cost**: 1000
- Cost is calculated from selected fields (connections cost more)
- Use `extensions.cost` in responses to monitor usage
- For bulk data, use [Bulk Operations](https://shopify.dev/docs/api/usage/bulk-operations/queries)

## Error Handling

- **200 + `errors` array**: schema/permission errors (check `extensions.code`)
- **200 + `userErrors` in mutation payload**: business-logic failures
- **`THROTTLED`**: reduce query complexity or wait for bucket refill
- **`ACCESS_DENIED`**: missing access scope — check app permissions
- **4xx/5xx**: network or account issues (rare)

## Quick Reference

For complete query/mutation examples, field lists, filter syntax, and input types:

- See [reference.md](reference.md) for embedded patterns
- See https://shopify.dev/docs/api/admin-graphql/2026-01 for full docs
- Run targeted introspection to discover fields not in reference.md

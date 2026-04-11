# Inventory & Locations

Patterns for inventory levels, stock adjustments, and locations.

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

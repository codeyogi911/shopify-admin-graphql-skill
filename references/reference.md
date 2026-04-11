# Shopify Admin GraphQL — Reference Index

This directory holds topic-scoped reference files. Open only the file(s) relevant to the task — loading all of them at once wastes context.

## Topic map

| File | Covers |
|---|---|
| [catalog.md](catalog.md) | Products, Collections, Metafields |
| [inventory.md](inventory.md) | Inventory levels & adjustments, Locations |
| [logistics.md](logistics.md) | Delivery Profiles, Shipping rates, Carrier Services, Markets |
| [sales.md](sales.md) | Orders, Discounts, Fulfillment |
| [customers.md](customers.md) | Customers (list, create, update) |
| [b2b.md](b2b.md) | Companies, Locations, Catalogs, Price Lists, Payment Terms (Plus only) |
| [marketing.md](marketing.md) | Customer Segments, Selling Plans, Gift Cards, Marketing Activities |
| [content.md](content.md) | Shop info, Blogs & Articles |
| [analytics.md](analytics.md) | ShopifyQL queries, reports, aggregates |
| [platform.md](platform.md) | Bulk operations, Webhooks, Access scopes |

## How to pick a file

- **"Get/update products, variants, collections, or metafields"** → `catalog.md`
- **"Stock levels, adjust inventory, locations"** → `inventory.md`
- **"Shipping rates, delivery zones, carrier rates, markets, international"** → `logistics.md`
- **"Orders, refunds, fulfillment, discount codes"** → `sales.md`
- **"Customer records, segments, addresses"** → `customers.md`
- **"B2B companies, wholesale pricing, payment terms, Net-30"** → `b2b.md`
- **"Customer segments, subscriptions, gift cards, marketing campaigns"** → `marketing.md`
- **"Blog posts, articles, shop config"** → `content.md`
- **"Sales reports, dashboard metrics, GROUP BY aggregates"** → `analytics.md`
- **"Bulk export, webhook subscriptions, required scopes"** → `platform.md`

## Extending the reference

When you discover a new useful pattern via introspection:

1. Pick the topic file that fits best (or propose a new one if none fit).
2. Append the pattern using the existing section format: short heading, GraphQL block, optional variables JSON, brief notes.
3. Keep examples minimal — only the fields a typical caller needs.

See [../SKILL.md](../SKILL.md) for introspection queries and API-wide rules.

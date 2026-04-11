# Shopify Admin GraphQL Skill

A reference skill for generating correct Shopify Admin GraphQL queries and mutations. Drop it into a Claude Code / Cursor agent and the agent gets topic-scoped patterns for products, orders, inventory, B2B, marketing, logistics, content, analytics, app billing, and more — all verified against the latest stable Shopify Admin API.

**Target API version**: `2026-04` (latest stable).

## What's in here

```
SKILL.md              Entry point — API basics, rules, introspection, rate limits
references/
  reference.md        Topic index — read this first to pick the right file
  catalog.md          Products, variants, collections, files, publications, metaobjects
  inventory.md        Inventory levels, locations
  sales.md            Orders, draft orders, returns, refunds, order editing, risk
  customers.md        Customer CRUD
  b2b.md              Companies, locations, catalogs, price lists, payment terms (Plus)
  marketing.md        Customer segments, selling plans, gift cards, marketing activities
  logistics.md        Delivery profiles, carrier services, markets, international
  content.md          Shop info, blogs, articles, pages, redirects, menus, translations, themes
  analytics.md        ShopifyQL reporting via shopifyqlQuery
  platform.md         Bulk ops, webhooks, scopes, app billing, staff, policies, events, Shopify Payments
```

Every topic file is self-contained. The agent loads the index first, picks the file(s) relevant to the task, and only pays the context cost of what it actually needs.

## How to use it

### In Claude Code / Cursor

Point the agent at the skill directory. When a user asks a Shopify question, the agent should:

1. Read `SKILL.md` for the API-wide rules (version, auth, rate limits, pagination, error handling).
2. Open `references/reference.md` and match the task to a topic file.
3. Read only that topic file (not all of them).
4. If the needed pattern isn't there, run the targeted introspection query from `SKILL.md` against a test store.

### Core rules the skill enforces

- Always use GraphQL **variables** — never interpolate user input into query strings.
- Always request **`userErrors { field message }`** in mutations. A 200 response ≠ success.
- Always include **`pageInfo { hasNextPage endCursor }`** on paginated queries.
- Prefer the **`query`** argument for server-side filtering over client-side filtering.
- Use full **GIDs** (`gid://shopify/Product/123`), never raw numeric IDs.
- Prefer **`productSet`** over `productUpdate` for multi-variant sync.
- **`refundCreate` requires `@idempotent(key:)`** as of API 2026-04.

## What's covered

Roughly 60 verified queries and mutations across the Admin API surface, plus ~25 enum listings and workflow explanations for the non-obvious parts:

- Idempotent refunds, two-step staged uploads, three-step redirect imports
- Order editing begin → stage → commit flow
- Translatable digest staleness invariant
- `themeFilesUpsert` exemption requirement
- App subscription confirmation URL flow
- Cumulative-vs-incremental marketing engagement semantics
- Payout status deprecations (`IN_TRANSIT` → use `SCHEDULED`/`PAID`/`FAILED`/`CANCELED`)
- Market regions migration (deprecated `marketRegionsCreate` → `marketUpdate` with `conditions`)

Each pattern was cross-checked against `shopify.dev` during writing. Honest caveats and "introspect if this errors" flags are noted inline where a specific field name wasn't individually verified.

## Extending the skill

When you find a pattern that should be in here:

1. Run targeted introspection (`IntrospectType` query in `SKILL.md`) to confirm the current field/argument shape.
2. Open the topic file that fits best (or propose a new one if none fit).
3. Append a new section matching the existing format: short heading → GraphQL block → optional variables JSON → brief notes on gotchas.
4. Keep examples minimal. Only the fields a typical caller needs.

Avoid expanding `SKILL.md` itself — it should stay a small set of API-wide rules.

## Authentication examples

See `SKILL.md` for cURL, Node.js (`@shopify/shopify-api`), and embedded-app (`shopify:admin/...`) authentication snippets.

## Known gaps

Not yet covered (candidates for future batches):

- Checkout customization via Shopify Functions (discount, shipping, payment, delivery functions)
- Order transactions / manual capture in depth
- Multi-location inventory routing via `FulfillmentOrder`
- Customer merge, SMS consent, tax exemptions (current `customers.md` is thin)
- `inventoryMoveQuantities`, `inventoryActivate`/`inventoryDeactivate`

PRs welcome for any of these.

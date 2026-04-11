# Catalog — Products, Collections, Metafields

Common GraphQL patterns for managing the product catalog.

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
- `title:hoodie` — title contains "hoodie" (Shopify search tokenizes, no `*…*` wildcards; `hood*` prefix is supported)
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
mutation UpdateProduct($product: ProductUpdateInput!) {
  productUpdate(product: $product) {
    product { id title }
    userErrors { field message }
  }
}
```

Note: the older `input: ProductInput` argument is deprecated. Use `product: ProductUpdateInput` (the `id` lives inside the input). For multi-variant updates, prefer `productSet`.

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

## productSet — full product sync in one call

Use `productSet` when you're syncing a product's full state from an external system (PIM, ERP, feed). Unlike `productUpdate`, for **list fields** (variants, collections, metafields) it creates new, updates existing, **and deletes anything not in the request**. Non-list fields behave like `productUpdate` — omitted fields stay unchanged.

**Required scope**: `write_products`.

```graphql
mutation SetProduct($input: ProductSetInput!, $synchronous: Boolean) {
  productSet(input: $input, synchronous: $synchronous) {
    product {
      id
      title
      variants(first: 100) { edges { node { id sku price inventoryQuantity } } }
    }
    productSetOperation {
      id
      status
    }
    userErrors { field message }
  }
}
```

**Variables example** (full sync of a 2-option, 4-variant product):

```json
{
  "synchronous": true,
  "input": {
    "title": "Merino Crew",
    "vendor": "Acme",
    "productType": "Knitwear",
    "status": "ACTIVE",
    "productOptions": [
      { "name": "Color", "values": [{ "name": "Charcoal" }, { "name": "Oat" }] },
      { "name": "Size",  "values": [{ "name": "S" }, { "name": "M" }] }
    ],
    "variants": [
      { "optionValues": [{ "name": "Charcoal", "optionName": "Color" }, { "name": "S", "optionName": "Size" }], "price": "89.00", "inventoryItem": { "sku": "MC-CHA-S" } },
      { "optionValues": [{ "name": "Charcoal", "optionName": "Color" }, { "name": "M", "optionName": "Size" }], "price": "89.00", "inventoryItem": { "sku": "MC-CHA-M" } },
      { "optionValues": [{ "name": "Oat",      "optionName": "Color" }, { "name": "S", "optionName": "Size" }], "price": "89.00", "inventoryItem": { "sku": "MC-OAT-S" } },
      { "optionValues": [{ "name": "Oat",      "optionName": "Color" }, { "name": "M", "optionName": "Size" }], "price": "89.00", "inventoryItem": { "sku": "MC-OAT-M" } }
    ]
  }
}
```

**`synchronous` flag**:
- `true` (default) — returns the updated `product` immediately. Safe for small products.
- `false` — returns a `ProductSetOperation`; poll its `status` until `COMPLETE`. Use for products with many variants or when you expect the call to exceed request timeouts.

To update an existing product, include `id` inside `input`, or use the optional `identifier` argument with a handle.

---

## Product variants — bulk create & update

Use these when you don't want `productSet`'s delete-by-omission behavior and only need to add or patch a few variants.

### Bulk create variants

```graphql
mutation BulkCreateVariants($productId: ID!, $variants: [ProductVariantsBulkInput!]!) {
  productVariantsBulkCreate(productId: $productId, variants: $variants) {
    productVariants {
      id
      sku
      price
      selectedOptions { name value }
    }
    userErrors { field message }
  }
}
```

The `strategy` argument (`ProductVariantsBulkCreateStrategy`, default `DEFAULT`) controls what happens to the product's existing standalone variant:

- `DEFAULT` — deletes the "Default Title" variant if it's the only variant; leaves real custom standalone variants alone.
- `PRESERVE_STANDALONE_VARIANT` — keeps the existing single variant (default or custom) as-is.
- `REMOVE_STANDALONE_VARIANT` — deletes the existing single variant regardless of whether it's the default.

### Bulk update variants

```graphql
mutation BulkUpdateVariants($productId: ID!, $variants: [ProductVariantsBulkInput!]!) {
  productVariantsBulkUpdate(productId: $productId, variants: $variants) {
    productVariants { id sku price }
    userErrors { field message }
  }
}
```

Each entry in `variants` must include the variant `id`. `productId` is passed at the mutation level, not inside each variant. Pass `allowPartialUpdates: true` if you want valid variant changes to persist even when some entries fail validation.

---

## Files & staged uploads

Uploading non-URL media (local images, videos, PDFs) is a two-step dance:

1. Call `stagedUploadsCreate` → Shopify returns a presigned upload URL + form parameters.
2. POST the file bytes to that URL out-of-band.
3. Call `fileCreate` with `originalSource` set to the `resourceUrl` you got back.

For files already hosted at a public URL you can skip step 1 and pass the URL directly to `fileCreate`.

**Required scope**: `write_files`.

### Step 1 — request a staged upload target

```graphql
mutation StageUpload($input: [StagedUploadInput!]!) {
  stagedUploadsCreate(input: $input) {
    stagedTargets {
      url
      resourceUrl
      parameters { name value }
    }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "input": [
    {
      "resource": "PRODUCT_IMAGE",
      "filename": "merino-crew-charcoal.jpg",
      "mimeType": "image/jpeg",
      "httpMethod": "POST",
      "fileSize": "482133"
    }
  ]
}
```

`resource` enum values include: `PRODUCT_IMAGE`, `COLLECTION_IMAGE`, `SHOP_IMAGE`, `VIDEO`, `MODEL_3D`, `FILE`, `URL_REDIRECT_IMPORT`. `fileSize` is required for `VIDEO` and `MODEL_3D`.

### Step 2 — upload bytes to the returned `url` (outside GraphQL)

POST the file body to `stagedTargets[0].url` as multipart/form-data. Include every `parameters[].name → value` pair in the form before the `file` field — they carry the temporary upload credentials.

### Step 3 — register the file in Shopify

```graphql
mutation CreateFile($files: [FileCreateInput!]!) {
  fileCreate(files: $files) {
    files {
      id
      alt
      fileStatus
      ... on MediaImage { image { url width height } }
      ... on Video { sources { url format mimeType } }
      ... on GenericFile { url }
    }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "files": [
    {
      "originalSource": "https://shopify-staged-uploads.storage.googleapis.com/.../merino-crew-charcoal.jpg",
      "contentType": "IMAGE",
      "alt": "Merino Crew in Charcoal, front"
    }
  ]
}
```

`contentType` enum: `IMAGE`, `VIDEO`, `EXTERNAL_VIDEO`, `MODEL_3D`, `FILE`. Batch limit: 250 files per call. Files process asynchronously — poll `fileStatus` (`UPLOADED` → `PROCESSING` → `READY` | `FAILED`).

---

## Publications (Sales Channels)

A publication is a sales channel: Online Store, POS, Google, Facebook, headless storefronts, etc. Publishing a product or collection to a publication makes it visible on that channel.

**Required scopes**: `read_publications` + `write_publications`.

### List publications

```graphql
query ListPublications($first: Int!) {
  publications(first: $first) {
    edges {
      node {
        id
        name
        autoPublish
        supportsFuturePublishing
        catalog { id }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

### Publish a product (or any Publishable) to one or more channels

```graphql
mutation PublishResource($id: ID!, $input: [PublicationInput!]!) {
  publishablePublish(id: $id, input: $input) {
    publishable {
      ... on Product { id title publishedOnCurrentPublication }
      ... on Collection { id title }
    }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "id": "gid://shopify/Product/123",
  "input": [
    { "publicationId": "gid://shopify/Publication/1" },
    { "publicationId": "gid://shopify/Publication/2", "publishDate": "2026-05-01T09:00:00Z" }
  ]
}
```

`publishDate` schedules future publication — only Online Store channels honor it; others publish immediately regardless. Resources implementing `Publishable` include `Product`, `Collection`, and (via the online-store app) pages.

Use `publishableUnpublish` with the same shape to remove a resource from a publication.

---

## Metaobjects

Metaobjects are custom structured records (lookbooks, FAQ entries, store locators, size guides). You first create a **definition** (the schema), then create **entries** that conform to it.

**Required scopes**: `write_metaobject_definitions` (schema) + `write_metaobjects` (entries).

### Create a metaobject definition (schema)

```graphql
mutation CreateMetaobjectDefinition($definition: MetaobjectDefinitionCreateInput!) {
  metaobjectDefinitionCreate(definition: $definition) {
    metaobjectDefinition {
      id
      type
      name
      fieldDefinitions { key name type { name } required }
    }
    userErrors { field message code }
  }
}
```

**Variables example**:

```json
{
  "definition": {
    "type": "lookbook",
    "name": "Lookbook",
    "fieldDefinitions": [
      { "key": "title",  "name": "Title",  "type": "single_line_text_field", "required": true },
      { "key": "season", "name": "Season", "type": "single_line_text_field" },
      { "key": "cover",  "name": "Cover",  "type": "file_reference" },
      { "key": "products", "name": "Products", "type": "list.product_reference" }
    ],
    "capabilities": { "publishable": { "enabled": true } }
  }
}
```

Prefix `type` with `$app:` (e.g. `$app:lookbook`) to reserve the definition to your app so other apps can't modify it.

### Create a metaobject entry

```graphql
mutation CreateMetaobject($metaobject: MetaobjectCreateInput!) {
  metaobjectCreate(metaobject: $metaobject) {
    metaobject {
      id
      handle
      type
      fields { key value type }
    }
    userErrors { field message code }
  }
}
```

**Variables example**:

```json
{
  "metaobject": {
    "type": "lookbook",
    "handle": "winter-2026",
    "fields": [
      { "key": "title",  "value": "Winter 2026" },
      { "key": "season", "value": "winter" },
      { "key": "products", "value": "[\"gid://shopify/Product/123\",\"gid://shopify/Product/456\"]" }
    ],
    "capabilities": { "publishable": { "status": "ACTIVE" } }
  }
}
```

List-type fields (like `list.product_reference`) take a JSON-encoded string, not a native array.

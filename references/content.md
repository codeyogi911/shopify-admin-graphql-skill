# Content — Shop Info, Blogs & Articles

Patterns for shop configuration and online-store content (blogs, articles).

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

## Blogs & Articles

### List blogs

```graphql
query ListBlogs($first: Int!, $after: String) {
  blogs(first: $first, after: $after) {
    edges {
      node {
        id
        title
        handle
        commentPolicy
        articlesCount { count }
        createdAt
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Sort keys**: `HANDLE`, `ID`, `TITLE`

### Get single blog with articles

```graphql
query GetBlog($id: ID!, $first: Int!) {
  blog(id: $id) {
    id
    title
    handle
    commentPolicy
    templateSuffix
    tags
    articles(first: $first) {
      edges {
        node {
          id
          title
          handle
          isPublished
          publishedAt
          author { name }
          tags
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

### Create blog

```graphql
mutation CreateBlog($blog: BlogCreateInput!) {
  blogCreate(blog: $blog) {
    blog {
      id
      title
      handle
      commentPolicy
    }
    userErrors { code field message }
  }
}
```

**Variables example**:

```json
{
  "blog": {
    "title": "Brew Guides",
    "handle": "brew-guides",
    "commentPolicy": "MODERATED"
  }
}
```

**CommentPolicy enum**: `AUTO_PUBLISHED`, `CLOSED`, `MODERATED`

### Update blog

```graphql
mutation UpdateBlog($id: ID!, $blog: BlogUpdateInput!) {
  blogUpdate(id: $id, blog: $blog) {
    blog { id title handle commentPolicy }
    userErrors { code field message }
  }
}
```

### Delete blog

```graphql
mutation DeleteBlog($id: ID!) {
  blogDelete(id: $id) {
    deletedBlogId
    userErrors { code field message }
  }
}
```

### List articles

```graphql
query ListArticles($first: Int!, $after: String, $query: String) {
  articles(first: $first, after: $after, query: $query) {
    edges {
      node {
        id
        title
        handle
        isPublished
        publishedAt
        author { name }
        blog { id title }
        tags
        summary
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples** (`query` variable):
- `blog_id:123` — articles in a specific blog
- `author:Jane` — by author name
- `tag:coffee` — by tag
- `published_status:published` / `published_status:unpublished`
- `created_at:>2024-01-01` — after date
- `title:espresso` — title contains "espresso" (no `*…*` wildcards; `espr*` prefix is supported)

**Sort keys**: `AUTHOR`, `BLOG_TITLE`, `ID`, `PUBLISHED_AT`, `TITLE`, `UPDATED_AT`

### Get single article

```graphql
query GetArticle($id: ID!) {
  article(id: $id) {
    id
    title
    handle
    body
    summary
    author { name }
    blog { id title }
    image { url altText }
    isPublished
    publishedAt
    tags
    createdAt
    updatedAt
    templateSuffix
    seo { title description }
  }
}
```

### Create article (publish a blog post)

```graphql
mutation CreateArticle($article: ArticleCreateInput!) {
  articleCreate(article: $article) {
    article {
      id
      title
      handle
      isPublished
      publishedAt
    }
    userErrors { code field message }
  }
}
```

**Variables example**:

```json
{
  "article": {
    "title": "How to Brew the Perfect V60",
    "author": { "name": "Fix Coffee" },
    "blogId": "gid://shopify/Blog/123",
    "body": "<h2>What You Need</h2><p>A V60 dripper, filters, freshly ground coffee...</p>",
    "summary": "A step-by-step guide to brewing with the Hario V60.",
    "handle": "how-to-brew-perfect-v60",
    "tags": ["brewing", "v60", "pour-over"],
    "isPublished": true,
    "publishDate": "2026-03-29T10:00:00Z",
    "image": {
      "url": "https://cdn.shopify.com/...",
      "altText": "V60 brewing setup"
    }
  }
}
```

You can also create a new blog inline by passing the `blog` argument instead of `blogId`:

```graphql
mutation CreateArticleWithNewBlog($article: ArticleCreateInput!, $blog: ArticleBlogInput) {
  articleCreate(article: $article, blog: $blog) {
    article { id title blog { id title } }
    userErrors { code field message }
  }
}
```

### Update article

```graphql
mutation UpdateArticle($id: ID!, $article: ArticleUpdateInput!) {
  articleUpdate(id: $id, article: $article) {
    article {
      id
      title
      handle
      isPublished
      publishedAt
    }
    userErrors { code field message }
  }
}
```

**Key fields**: `title`, `body`, `summary`, `handle`, `author`, `blogId` (move to different blog), `image`, `isPublished`, `publishDate`, `tags`, `templateSuffix`, `metafields`, `redirectNewHandle` (creates URL redirect from old handle).

### Delete article

```graphql
mutation DeleteArticle($id: ID!) {
  articleDelete(id: $id) {
    deletedArticleId
    userErrors { code field message }
  }
}
```

**Required scopes**: All blog and article mutations need `write_content` or `write_online_store_pages`.

---

## Pages

A `Page` is a static content page on the online store (About, Contact, Terms, etc.). Separate from articles — pages aren't tied to a blog and have no timeline/feed semantics.

**Required scopes**: `write_content` or `write_online_store_pages`.

### List pages

```graphql
query ListPages($first: Int!, $after: String, $query: String) {
  pages(first: $first, after: $after, query: $query) {
    edges {
      node {
        id
        title
        handle
        isPublished
        publishedAt
        templateSuffix
        bodySummary
        createdAt
        updatedAt
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `title:about`, `handle:contact`, `published_status:published`, `created_at:>2026-01-01`

**Sort keys**: `ID`, `TITLE`, `UPDATED_AT`, `PUBLISHED_AT`, `RELEVANCE`

### Get single page

```graphql
query GetPage($id: ID!) {
  page(id: $id) {
    id
    title
    handle
    body
    bodySummary
    isPublished
    publishedAt
    templateSuffix
    author { name }
    seo { title description }
    metafields(first: 10) { edges { node { namespace key value type } } }
  }
}
```

### Create page

```graphql
mutation CreatePage($page: PageCreateInput!) {
  pageCreate(page: $page) {
    page { id title handle isPublished publishedAt }
    userErrors { code field message }
  }
}
```

**Variables example**:

```json
{
  "page": {
    "title": "Shipping & Returns",
    "handle": "shipping-returns",
    "body": "<h2>Domestic Shipping</h2><p>Orders ship within 2 business days...</p>",
    "isPublished": true,
    "templateSuffix": "contact"
  }
}
```

### Update / delete page

```graphql
mutation UpdatePage($id: ID!, $page: PageUpdateInput!) {
  pageUpdate(id: $id, page: $page) {
    page { id title handle }
    userErrors { code field message }
  }
}

mutation DeletePage($id: ID!) {
  pageDelete(id: $id) {
    deletedPageId
    userErrors { code field message }
  }
}
```

---

## URL Redirects

A `UrlRedirect` maps an old storefront path to a new target. Use for SEO-preserving migrations, renamed products/collections, and sunset campaigns.

**Required scope**: `write_online_store_navigation`.

### Create a single redirect

```graphql
mutation CreateRedirect($urlRedirect: UrlRedirectInput!) {
  urlRedirectCreate(urlRedirect: $urlRedirect) {
    urlRedirect { id path target }
    userErrors { field message }
  }
}
```

**Variables example**:

```json
{
  "urlRedirect": {
    "path": "/products/old-sku",
    "target": "/products/new-sku"
  }
}
```

Paths are storefront-relative (start with `/`). `target` can be relative or absolute.

### List redirects

```graphql
query ListRedirects($first: Int!, $query: String) {
  urlRedirects(first: $first, query: $query) {
    edges { node { id path target } }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Filter examples**: `path:/products`, `target:/collections`

### Bulk import redirects from a CSV

For migrations with hundreds or thousands of redirects, the single mutation is too slow. Use the three-step bulk import flow:

1. **`stagedUploadsCreate`** with `resource: URL_REDIRECT_IMPORT` — get a presigned upload URL.
2. **POST the CSV** to the staged URL (same out-of-band flow as product images; see [catalog.md](catalog.md#files--staged-uploads)).
3. **`urlRedirectImportCreate`** with the staged `resourceUrl`, then **`urlRedirectImportSubmit`** to process.

```graphql
mutation CreateRedirectImport($url: URL!) {
  urlRedirectImportCreate(url: $url) {
    urlRedirectImport {
      id
      count
      finished
    }
    userErrors { field message }
  }
}

mutation SubmitRedirectImport($id: ID!) {
  urlRedirectImportSubmit(id: $id) {
    job { id done }
    userErrors { field message }
  }
}
```

**CSV format**: two columns, `Redirect from,Redirect to`, one row per redirect, header row required. Shopify provides a sample template.

---

## Navigation Menus

A `Menu` is a navigation tree: main menu, footer menu, or any custom menu rendered by the theme. Items can link to products, collections, pages, blogs, articles, or arbitrary URLs, and nest up to **three levels deep**.

**Required scope**: `write_online_store_navigation`.

### List menus

```graphql
query ListMenus($first: Int!) {
  menus(first: $first) {
    edges {
      node {
        id
        handle
        title
        items {
          id
          title
          url
          type
          resourceId
          items {
            id
            title
            url
            items { id title url }
          }
        }
      }
    }
  }
}
```

### Create a menu

`menuCreate` takes the fields as **top-level scalar arguments** (not wrapped in an input object):

```graphql
mutation CreateMenu($title: String!, $handle: String!, $items: [MenuItemCreateInput!]!) {
  menuCreate(title: $title, handle: $handle, items: $items) {
    menu {
      id
      handle
      title
      items { id title url type items { id title url } }
    }
    userErrors { field message code }
  }
}
```

**Variables example** (a main menu with a top-level "Shop" entry containing two sub-collections):

```json
{
  "title": "Main menu",
  "handle": "main-menu",
  "items": [
    { "title": "Home", "type": "FRONTPAGE", "url": "/" },
    {
      "title": "Shop",
      "type": "COLLECTION",
      "resourceId": "gid://shopify/Collection/111",
      "items": [
        { "title": "New Arrivals", "type": "COLLECTION", "resourceId": "gid://shopify/Collection/222" },
        { "title": "Sale",         "type": "COLLECTION", "resourceId": "gid://shopify/Collection/333" }
      ]
    },
    { "title": "About",   "type": "PAGE",    "resourceId": "gid://shopify/Page/444" },
    { "title": "Contact", "type": "HTTP",    "url": "mailto:hello@example.com" }
  ]
}
```

**`MenuItemType` enum** (common values): `FRONTPAGE`, `CATALOG`, `COLLECTION`, `COLLECTIONS`, `PRODUCT`, `BLOG`, `ARTICLE`, `PAGE`, `SEARCH`, `CUSTOMER_ACCOUNT_PAGE`, `SHOP_POLICY`, `HTTP` (arbitrary URL).

For `COLLECTION`/`PRODUCT`/`PAGE`/`BLOG`/`ARTICLE` items, pass `resourceId`. For `HTTP` items, pass `url`. Don't pass both — Shopify rejects the call.

### Update / delete a menu

```graphql
mutation UpdateMenu($id: ID!, $title: String, $handle: String, $items: [MenuItemUpdateInput!]) {
  menuUpdate(id: $id, title: $title, handle: $handle, items: $items) {
    menu { id title handle }
    userErrors { field message code }
  }
}

mutation DeleteMenu($id: ID!) {
  menuDelete(id: $id) {
    deletedMenuId
    userErrors { field message code }
  }
}
```

---

## Translations

Shopify stores every translatable string as a `(resourceId, key, locale)` triple. To add a translation:

1. Query **`translatableResources`** or **`translatableResource`** to get the list of translatable fields on the resource, including a **digest** hash for each.
2. Call **`translationsRegister`** with the `(key, locale, value, translatableContentDigest)` for each string you're translating.

The digest is a fingerprint of the source content — if the source changes after you registered the translation, the digest becomes stale and Shopify marks the translation outdated. You must re-fetch the current digest and re-register whenever the source text changes.

**Required scopes**: `read_translations` + `write_translations`.

### List translatable content for a resource type

```graphql
query ListTranslatable($resourceType: TranslatableResourceType!, $first: Int!) {
  translatableResources(resourceType: $resourceType, first: $first) {
    edges {
      node {
        resourceId
        translatableContent {
          key
          value
          digest
          locale
        }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**`TranslatableResourceType` enum** (common values): `PRODUCT`, `COLLECTION`, `PAGE`, `BLOG`, `ARTICLE`, `METAFIELD`, `ONLINE_STORE_THEME`, `SHOP`, `LINK`, `FILTER`, `PAYMENT_GATEWAY`, `DELIVERY_METHOD_DEFINITION`.

### Get translatable fields for a specific resource

```graphql
query GetTranslatable($resourceId: ID!) {
  translatableResource(resourceId: $resourceId) {
    resourceId
    translatableContent { key value digest locale }
    translations(locale: "fr") { key value locale market { id handle } outdated }
  }
}
```

### Register translations

```graphql
mutation RegisterTranslations($resourceId: ID!, $translations: [TranslationInput!]!) {
  translationsRegister(resourceId: $resourceId, translations: $translations) {
    translations { key value locale market { id handle } }
    userErrors { field message code }
  }
}
```

**Variables example** (translate a product title and description to French):

```json
{
  "resourceId": "gid://shopify/Product/123",
  "translations": [
    {
      "locale": "fr",
      "key": "title",
      "value": "Pull col rond en mérinos",
      "translatableContentDigest": "abc123...the-digest-from-translatableResource"
    },
    {
      "locale": "fr",
      "key": "body_html",
      "value": "<p>Doux, chaud, et fabriqué à partir de laine mérinos...</p>",
      "translatableContentDigest": "def456...the-digest"
    }
  ]
}
```

Add an optional `marketId` inside each `TranslationInput` to scope the translation to one market only (e.g., French for France but a different French for Canada).

### Remove translations

```graphql
mutation RemoveTranslations($resourceId: ID!, $translationKeys: [String!]!, $locales: [String!]!) {
  translationsRemove(resourceId: $resourceId, translationKeys: $translationKeys, locales: $locales) {
    translations { key locale }
    userErrors { field message code }
  }
}
```

---

## Themes & Theme Files

Read the list of installed themes and (with a Shopify exemption) push updates to theme files via `themeFilesUpsert`.

**Required scopes**: `read_themes` / `write_themes`.

> ⚠️ **`themeFilesUpsert` requires a special exemption from Shopify** on top of the `write_themes` scope. General third-party apps cannot call it. For most app-facing theme customization, use **theme app extensions** + app blocks instead. If you're building an internal tool for a single shop, contact Partner support to request the exemption.

### List themes

```graphql
query ListThemes($first: Int!, $roles: [ThemeRole!]) {
  themes(first: $first, roles: $roles) {
    edges {
      node {
        id
        name
        role
        processing
        createdAt
        updatedAt
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

**`ThemeRole` enum**: `MAIN` (the published/live theme), `UNPUBLISHED` (drafts), `DEVELOPMENT` (dev/preview), `MOBILE` (mobile-specific variant), `DEMO` (showroom).

Filter to the live theme: `{"roles": ["MAIN"]}`. Filter to all drafts: `{"roles": ["UNPUBLISHED"]}`.

### Read theme files

```graphql
query GetThemeFiles($id: ID!, $first: Int!, $filenames: [String!]) {
  theme(id: $id) {
    id
    name
    role
    files(first: $first, filenames: $filenames) {
      edges {
        node {
          filename
          size
          contentType
          body {
            ... on OnlineStoreThemeFileBodyText { content }
            ... on OnlineStoreThemeFileBodyBase64 { contentBase64 }
            ... on OnlineStoreThemeFileBodyUrl { url }
          }
          updatedAt
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}
```

Pass `filenames` to fetch a specific subset (e.g., `["config/settings_data.json", "sections/header.liquid"]`) instead of paginating through everything.

### Upsert theme files (exemption required)

```graphql
mutation UpsertThemeFiles($themeId: ID!, $files: [OnlineStoreThemeFilesUpsertFileInput!]!) {
  themeFilesUpsert(themeId: $themeId, files: $files) {
    upsertedThemeFiles { filename }
    job { id done }
    userErrors { filename code message }
  }
}
```

**Variables example** (update a settings JSON file and a Liquid section):

```json
{
  "themeId": "gid://shopify/OnlineStoreTheme/789",
  "files": [
    {
      "filename": "config/settings_data.json",
      "body": {
        "type": "TEXT",
        "value": "{\"current\":\"Default\",\"presets\":{}}"
      }
    },
    {
      "filename": "sections/custom-banner.liquid",
      "body": {
        "type": "TEXT",
        "value": "<div class=\"banner\">{{ section.settings.heading }}</div>"
      }
    }
  ]
}
```

**Batch limit**: 50 files per call.

**`OnlineStoreThemeFileBodyInputType` enum**: `TEXT` (inline string), `BASE64` (binary assets), `URL` (fetch from a remote URL — useful for large media you've already uploaded elsewhere).

The mutation is asynchronous — `job.done` is `false` initially. Poll the `Job` via the `job(id:)` query until `done: true`.

### Delete theme files

```graphql
mutation DeleteThemeFiles($themeId: ID!, $files: [String!]!) {
  themeFilesDelete(themeId: $themeId, files: $files) {
    deletedThemeFiles { filename }
    userErrors { filename code message }
  }
}
```

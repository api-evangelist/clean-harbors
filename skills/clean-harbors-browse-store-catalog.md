---
name: clean-harbors-browse-store-catalog
description: >-
  Query the Safety-Kleen / Clean Harbors online store catalog — products, prices and
  categories — over the anonymous GraphQL endpoint, without an account or a token.
generated: '2026-09-05'
method: generated
source: >-
  Grounded in graphql/clean-harbors-store-schema.graphql, introspected live from
  https://store.safety-kleen.com/graphql on 2026-09-05. Every field named below exists in
  that schema, and the example queries were executed against the live endpoint and returned
  HTTP 200.
api: Clean Harbors / Safety-Kleen commerce GraphQL
base: https://store.safety-kleen.com/graphql
operations:
  - products
  - categories
  - storeConfig
  - availableStores
  - cmsPage
  - route
---

# Browse the Safety-Kleen store catalog

`https://store.safety-kleen.com/graphql` is a live Adobe Commerce (Magento 2) GraphQL
endpoint operated by Clean Harbors. The catalog half of it answers **anonymously** — no key,
no token, no cookie. Introspection is enabled, so the full 861-type contract is readable at
runtime.

The same instance serves six Clean Harbors brand storefronts. `storeConfig.website_name`
returns `"Clean Harbors USA Website"`, and `availableStores` lists
`store.safety-kleen.com`, `store.thermofluids.com`, `store.nobleoil.com`,
`store.emeraldrenews.com`, `store.murphyswasteoil.com` and
`store.synergyrecycling.org`. Send a `Store: <store_code>` header to target a specific
brand; without it you get `en_us` (Safety-Kleen USA).

## Steps

1. **Confirm which storefront you are on.**

       { storeConfig { store_code store_name website_name base_url base_currency_code } }

2. **Search products.** `products` takes `search`, `filter`, `sort`, `pageSize` and
   `currentPage`.

       {
         products(search: "oil", pageSize: 20) {
           total_count
           page_info { current_page page_size total_pages }
           items {
             sku
             name
             url_key
             price_range { minimum_price { final_price { value currency } } }
           }
         }
       }

   Executed on 2026-09-05 this returned `total_count: 277`.

3. **Page by number.** `page_info` is a `SearchResultPageInfo`
   (`current_page`, `page_size`, `total_pages`). There is **no cursor pagination anywhere in
   this schema**, so a long walk cannot be resumed from an opaque token — track
   `currentPage` yourself.

4. **Walk categories with `categories`, not `categoryList`.** `Query.category` and
   `Query.categoryList` are both `@deprecated` in favour of `categories`.

       { categories(pageSize: 20) { total_count items { uid name url_path product_count } } }

5. **Resolve a storefront URL to what it points at** with `route(url: "…")`.
   `Query.urlResolver` is deprecated in its favour.

## What needs a token

Anything customer-scoped — `customer`, `customerCart`, `company`, `negotiableQuotes`,
requisition lists, purchase orders, wishlists — needs an `Authorization: Bearer <token>`
header. The token comes from the `generateCustomerToken` mutation and is revoked with
`revokeCustomerToken`. Clean Harbors documents no way for a third party to obtain store
credentials, so treat the authenticated half as out of reach unless you are already a
customer.

The parallel REST surface at `https://store.safety-kleen.com/rest/` is almost entirely
Bearer-gated (`{"message":"Missing Bearer token."}`, HTTP 401) and its Swagger generator at
`/rest/all/schema?services=all` times out with HTTP 503, so there is no REST contract to work
from. Use GraphQL.

## Before you write anything

Read `conventions/clean-harbors-conventions.yml` first if you intend to call a mutation.
Two facts matter:

- **There is no idempotency mechanism.** None of the 168 mutations — `placeOrder`,
  `placePurchaseOrder` and `placeNegotiableQuoteOrder` included — accepts an idempotency key
  or a client request token. A retried write can duplicate.
- **Reversal paths exist but no window is published.** `cancelOrder`, `confirmCancelOrder`,
  `requestReturn` and `cancelPurchaseOrders` are in the contract, but Clean Harbors publishes
  no returns or cancellation policy for this store (`/return-policy`, `/returns`,
  `/shipping-returns` and `/terms-and-conditions` all 404), so how long you have to undo an
  action is unknown. Do not assume one.

## Errors

Field errors come back as a GraphQL `errors[]` array with HTTP `200`; a malformed query is
HTTP `400`. Typed error enums (`CartUserInputErrorType`, `PurchaseOrderErrorType`,
`ClearCartErrorType` and others) are catalogued in
`errors/clean-harbors-problem-types.yml`. Nothing here uses RFC 9457 problem details.

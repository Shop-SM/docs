# InlineCategoryProducts — GraphQL Query Reference

> **Location:** `lib/widgets/store_list_selector_overlay.dart` · Line 422  
> **Widget:** `InlineCategoryProducts`  
> **Query:** `unbxdProducts`

---

## Table of Contents

- [Overview](#overview)
- [Data Flow](#data-flow)
- [Widget Usage](#widget-usage)
- [Provider Selection](#provider-selection)
- [Repository Call](#repository-call)
- [GraphQL Query](#graphql-query)
- [Variables](#variables)
- [HTTP Request Details](#http-request-details)
  - [Endpoint](#endpoint)
  - [Headers](#headers)
- [Fetch Policy](#fetch-policy)
- [Source Files](#source-files)

---

## Overview

When the CMS (Prismic) returns a body slice of type `"inline_category"` with a valid `categoryId`, the store list overlay renders an `InlineCategoryProducts` widget. This widget fetches a horizontal product carousel from the **Magento GraphQL API** using the `unbxdProducts` query, scoped to the user's currently selected branch via the `Store` HTTP header.

---

## Data Flow

```
StoreListSelectorOverlay  (L414–L431)
  └── _buildBodySlivers()  →  case "inline_category"
        └── InlineCategoryProducts widget
              ├── (with stores)   categoryProductsWithStoreFutureProvider
              └── (no stores)     categoryProductsFutureProvider
                    └── StorefrontRepositoryImpl.getProducts()
                          └── unbxdProducts  →  GET {marketsDomain}/graphql
```

---

## Widget Usage

```dart
// lib/widgets/store_list_selector_overlay.dart — _buildBodySlivers()
case "inline_category":
  final fallbackStores = item.primary?.fallbackDefaultBranchCodes
      ?.split(',')
      .map((e) => e.trim())
      .where((e) => e.isNotEmpty)
      .toList();

  return item.primary?.categoryId != null
      ? InlineCategoryProducts(
          id: "${item.type}-${i++}",
          bgColor: item.primary?.backgroundColor,
          textColor: item.primary?.textColor,
          title: item.primary?.title1?.first.text,
          categoryId: item.primary!.categoryId!,   // required
          stores: fallbackStores,                   // optional branch codes from CMS
          isUniversal: true,
        )
      : null;
```

---

## Provider Selection

The widget picks a provider based on whether fallback store codes are available.

```dart
// lib/widgets/inline_category_products.dart
final categoryProductsFuture = (stores != null && stores!.isNotEmpty)
    ? ref.watch(
        categoryProductsWithStoreFutureProvider(Tuple2(categoryId, stores)),
      )
    : ref.watch(categoryProductsFutureProvider(categoryId));
```

### `categoryProductsFutureProvider`
Used when **no fallback stores** are provided. Fetches products without a `stores` filter.

```dart
final categoryProductsFutureProvider = FutureProvider.autoDispose
    .family<List<Product>, String>((ref, categoryId) async {
  final repository = ref.watch(storefrontRepository);
  return repository.getProducts(categoryId: categoryId);
});
```

### `categoryProductsWithStoreFutureProvider`
Used when fallback store codes are present. Prefers the user's **active branch codes** over the CMS fallback.

```dart
final categoryProductsWithStoreFutureProvider = FutureProvider.autoDispose
    .family<List<Product>, Tuple2<String, List<String>?>>((ref, params) async {
  final categoryId = params.item1;
  final stores     = params.item2;

  final repository   = ref.watch(storefrontRepository);
  final branchCodes  = ref.watch(branchCodesProvider); // user's active branches

  return repository.getProducts(
    categoryId: categoryId,
    stores: (branchCodes.isNotEmpty) ? branchCodes : stores,
    // Priority: active branch codes > CMS fallback codes
  );
});
```

> **Store resolution priority**
> 1. User's selected branch codes (`branchCodesProvider`) — highest priority
> 2. `fallbackDefaultBranchCodes` from the Prismic CMS slice
> 3. Omitted entirely — returns results across all stores

---

## Repository Call

```dart
// lib/views/storefront/repositories/storefront_repository.dart
Future<List<Product>> getProducts({
  num page = 1,
  num pageSize = 24,
  required String categoryId,
  List<String>? stores,
}) async {
  final QueryOptions options = QueryOptions(
    document: gql(getProductsQuery),
    variables: {
      if (stores != null && stores.isNotEmpty) "stores": stores,
      "pageSize": pageSize.toString(),
      "currentPage": page.toString(),
      "filter": {
        "category_id": {"eq": categoryId},
      },
    },
    fetchPolicy: FetchPolicy.networkOnly,
  );

  final QueryResult result = await _client.query(options);
  // Parses result.data['unbxdProducts']['items'] → List<Product>
}
```

---

## GraphQL Query

**Operation name:** `unbxdProducts`  
**Defined in:** `lib/gql/magento.dart` · `getProductsQuery`

```graphql
query unbxdProducts(
  $filter: ProductAttributeFilterInput,
  $pageSize: Int,
  $currentPage: Int,
  $search: String,
  $stores: [String]
) {
  unbxdProducts(
    search: $search,
    filter: $filter,
    pageSize: $pageSize,
    currentPage: $currentPage,
    stores: $stores
  ) {
    items {
      id
      uom
      weighted_price
      description { html __typename }
      stock_status
      product_label {
        label_id
        label_description
        label_name
        label_status
        label_from_date
        label_to_date
        label_priority
        label_type
        picture_frame { position type url }
        stores
        customer_groups
        category_image {
          type url position display text text_color text_font
          text_size shape_type label_size label_size_mobile custom_css
          position __typename
        }
      }
      max_qty
      special_price
      special_from_date
      special_to_date
      media_gallery {
        ... on ProductImage { disabled position url label }
      }
      meta_description
      name
      small_image { url __typename }
      sku
      url_key
      product_link
      promo_text
      recommended_product
      nearest_store
      size_chart
    }
    request_id
  }
}
```

> **Note:** Uses the `productFieldsSimplified` fragment — omits `price`, `price_range`, and `storesAvailable` compared to the full `productFields` fragment used in the product detail query.

---

## Variables

| Variable       | Type                          | Default  | Example                                    | Notes                                         |
|----------------|-------------------------------|----------|--------------------------------------------|-----------------------------------------------|
| `filter`       | `ProductAttributeFilterInput` | —        | `{ "category_id": { "eq": "42" } }`        | Always present; `categoryId` comes from CMS   |
| `pageSize`     | `Int` (sent as string)        | `"24"`   | `"24"`                                     | Not exposed to the widget; always 24          |
| `currentPage`  | `Int` (sent as string)        | `"1"`    | `"1"`                                      | No pagination; always page 1                  |
| `stores`       | `[String]`                    | omitted  | `["SMSM", "SMNO"]`                         | Optional; omitted when no branch codes exist  |
| `search`       | `String`                      | omitted  | —                                           | Not used in this flow                         |

---

## HTTP Request Details

### Endpoint

```
GET {marketsDomain}/graphql?query=...&variables=...
```

Requests are sent via **HTTP GET** because `useGETForQueries: true` is set on `DioLink`.

| Environment  | Base URL                               |
|--------------|----------------------------------------|
| **PROD**     | `https://smmarkets.ph`                 |
| **UAT**      | `https://mcstaging2.smmarkets.ph`      |
| **STAGING**  | `https://mcstaging.smmarkets.ph`       |
| **DEV**      | `https://mcstaging.smmarkets.ph`       |

Active environment is controlled by `Config().env` in `lib/config.dart` (currently `Env.PROD`).

### Headers

| Header         | Value                                        | Source                                             |
|----------------|----------------------------------------------|----------------------------------------------------|
| `Store`        | `{branch.code}` · fallback: `"default"`      | `graphqlClientMagentoProvider` → `branchNotifierProvider` |
| `Content-Type` | `application/json`                           | Set automatically by `gql_dio_link` / Dio          |

**Client setup:**

```dart
// lib/app/providers/graphql_client_magento_provider.dart
final graphqlClientMagentoProvider = Provider<GraphQLClient>((ref) {
  final dio    = Dio();
  final branch = ref.watch(branchNotifierProvider);

  return GraphQLClient(
    link: DioLink(
      '${Config().variables?.marketsDomain}/graphql',
      client: dio,
      defaultHeaders: {"Store": branch?.code ?? "default"}, // ← Store header
      useGETForQueries: true,
    ),
    cache: GraphQLCache(store: HiveStore()),
    queryRequestTimeout: Duration(seconds: 15),
  );
});
```

> The provider watches `branchNotifierProvider`, so any branch change automatically creates a new `GraphQLClient` with the updated `Store` header — no manual refresh needed.

---

## Fetch Policy

`FetchPolicy.networkOnly` — **no cache reads or writes**. Every widget render triggers a fresh network request.

---

## Source Files

| File | Role |
|------|------|
| [`store_list_selector_overlay.dart` L414–L431](../lib/widgets/store_list_selector_overlay.dart) | Widget instantiation & CMS slice parsing |
| [`inline_category_products.dart`](../lib/widgets/inline_category_products.dart) | Widget definition & provider selection |
| [`category_products_future_provider.dart`](../lib/views/storefront/providers/category_products_future_provider.dart) | Riverpod async providers |
| [`storefront_repository.dart`](../lib/views/storefront/repositories/storefront_repository.dart) | Repository & query execution |
| [`magento.dart`](../lib/gql/magento.dart) | Raw GraphQL query strings |
| [`graphql_client_magento_provider.dart`](../lib/app/providers/graphql_client_magento_provider.dart) | HTTP client configuration & headers |
| [`config.dart`](../lib/config.dart) | Environment base URLs |
